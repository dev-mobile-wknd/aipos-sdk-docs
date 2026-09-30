# Speech-to-text

SDK sengaja tidak membawa mesin pengenal suara. Mikrofon, izin, dan siklus hidup layar
adalah wilayah aplikasi Anda. Tugas aplikasi hanya dua: ubah suara jadi teks, lalu setor ke
SDK.

| Kejadian dari pengenal suara | Panggil |
|---|---|
| Hasil sementara (kalimat belum selesai) | `sdk.setInterimTranscript(teks)` |
| Hasil final (kalimat selesai) | `sdk.pushTranscript(teks)` |

## Android — `SpeechRecognizer`

### Manifest

```xml
<uses-permission android:name="android.permission.RECORD_AUDIO" />

<!-- Required since Android 11: without this, isRecognitionAvailable() is always false -->
<queries>
    <intent>
        <action android:name="android.speech.RecognitionService" />
    </intent>
</queries>
```

Minta izin `RECORD_AUDIO` saat runtime sebelum mulai mendengarkan.

### Pengontrol

`SpeechRecognizer` berhenti setiap kali satu ujaran selesai. Untuk mendengarkan terus-menerus,
nyalakan ulang di `onResults` dan pada error "tidak ada suara":

```kotlin
class ConversationListener(
    context: Context,
    private val sdk: AIPosSDK,
) : RecognitionListener {

    private val recognizer = SpeechRecognizer.createSpeechRecognizer(context)
    private val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
        putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
        putExtra(RecognizerIntent.EXTRA_LANGUAGE, "id-ID")
        putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, true)
    }
    private var isActive = false

    init { recognizer.setRecognitionListener(this) }

    fun start() { isActive = true; recognizer.startListening(intent) }

    fun stop() { isActive = false; recognizer.stopListening() }

    fun release() { isActive = false; recognizer.destroy() }

    override fun onPartialResults(partialResults: Bundle) {
        partialResults.firstText()?.let(sdk::setInterimTranscript)
    }

    override fun onResults(results: Bundle) {
        results.firstText()?.takeIf { it.isNotBlank() }?.let(sdk::pushTranscript)
        sdk.setInterimTranscript("")
        if (isActive) recognizer.startListening(intent)
    }

    override fun onError(error: Int) {
        val canRetry = error == SpeechRecognizer.ERROR_NO_MATCH ||
            error == SpeechRecognizer.ERROR_SPEECH_TIMEOUT
        if (isActive && canRetry) recognizer.startListening(intent)
    }

    private fun Bundle.firstText(): String? =
        getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull()

    override fun onReadyForSpeech(params: Bundle?) = Unit
    override fun onBeginningOfSpeech() = Unit
    override fun onRmsChanged(rmsdB: Float) = Unit
    override fun onBufferReceived(buffer: ByteArray?) = Unit
    override fun onEndOfSpeech() = Unit
    override fun onEvent(eventType: Int, params: Bundle?) = Unit
}
```

`SpeechRecognizer` wajib dibuat dan dipanggil dari thread utama.

## iOS — `SFSpeechRecognizer`

### Info.plist

```xml
<key>NSMicrophoneUsageDescription</key>
<string>The microphone is used to listen to conversations with customers.</string>
<key>NSSpeechRecognitionUsageDescription</key>
<string>Conversations are converted to text so the AI can suggest products.</string>
```

### Menentukan akhir kalimat

`SFSpeechAudioBufferRecognitionRequest` pada mode *streaming* sering **tidak pernah**
menandai hasil sebagai final selama audio terus mengalir. Jangan menunggu `isFinal`; anggap
kalimat selesai setelah jeda hening singkat:

```swift
private var silenceTimer: Timer?
private let silenceDelay: TimeInterval = 1.2

func handle(_ result: SFSpeechRecognitionResult) {
    let text = result.bestTranscription.formattedString
    sdk.setInterimTranscript(text: text)

    silenceTimer?.invalidate()
    silenceTimer = Timer.scheduledTimer(withTimeInterval: silenceDelay, repeats: false) { [weak self] _ in
        guard let self, !text.isEmpty else { return }
        self.sdk.pushTranscript(text: text)
        self.sdk.setInterimTranscript(text: "")
        self.restartRecognitionSession()   // the next result starts from an empty sentence
    }
}
```

Mulai ulang sesi pengenalan setelah setiap kalimat final. Tanpa itu, `formattedString`
terus memanjang dan kalimat yang sama tersetor berulang kali.

## Tips

- **Pakai bahasa `id-ID`.** Model bahasa di SDK memahami campuran Indonesia–Inggris, tetapi
  pengenal suara yang disetel ke bahasa lain menghasilkan transkrip yang kacau.
- **Setor kalimat utuh, bukan kata per kata.** Hasil sementara cukup lewat
  `setInterimTranscript`.
- **Tidak perlu membatasi frekuensi.** SDK menunggu percakapan mereda sebelum menganalisis.
- **Hentikan mikrofon saat layar ditutup** dan panggil `clearAdvisor()` saat pelanggan berganti.
