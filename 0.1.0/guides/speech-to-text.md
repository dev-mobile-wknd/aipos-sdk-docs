# Speech-to-text

The SDK intentionally doesn't ship a speech-recognition engine. The microphone,
permissions, and screen lifecycle are your app's territory. The app's job is just
two things: turn speech into text, then feed it to the SDK.

| Event from the recognizer | Call |
|---|---|
| Interim result (sentence not finished) | `sdk.setInterimTranscript(text)` |
| Final result (sentence finished) | `sdk.pushTranscript(text)` |

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

Request the `RECORD_AUDIO` permission at runtime before you start listening.

### Controller

`SpeechRecognizer` stops every time one utterance finishes. For continuous
listening, restart it in `onResults` and on a "no speech" error:

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

`SpeechRecognizer` must be created and called from the main thread.

## iOS — `SFSpeechRecognizer`

### Info.plist

```xml
<key>NSMicrophoneUsageDescription</key>
<string>The microphone is used to listen to conversations with customers.</string>
<key>NSSpeechRecognitionUsageDescription</key>
<string>Conversations are converted to text so the AI can suggest products.</string>
```

### Determining the end of a sentence

`SFSpeechAudioBufferRecognitionRequest` in *streaming* mode often **never** marks
a result as final while audio keeps flowing. Don't wait for `isFinal`; treat a
sentence as done after a brief pause in silence:

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

Restart the recognition session after every final sentence. Without this,
`formattedString` keeps growing and the same sentence gets submitted repeatedly.

## Tips

- **Use the `id-ID` locale.** The SDK's language model understands a mix of
  Indonesian and English, but a recognizer set to another locale produces messy
  transcripts.
- **Submit whole sentences, not word by word.** Interim results are enough via
  `setInterimTranscript`.
- **No need to throttle.** The SDK waits for the conversation to settle before
  analyzing.
- **Stop the microphone when the screen is closed**, and call `clearAdvisor()`
  when the customer changes.
