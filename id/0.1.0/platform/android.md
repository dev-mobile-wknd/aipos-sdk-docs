# Integrasi Android

Halaman ini merangkum semua yang khusus Android: siklus hidup, terminal pembayaran,
lisensi MineSec, dan praktik terbaik dengan Jetpack Compose.

## Langkah wajib

Dua langkah terikat pada siklus hidup Android dan tidak bisa dilewati bila Anda memakai
pembayaran kartu (`aipos-payment` atau `aipos-sdk`).

### 1. `initPlatform` di `Application.onCreate`

MineSec memvalidasi lisensi dan menyiapkan kunci kriptografi sebelum transaksi pertama.
Mulailah sedini mungkin:

```kotlin
class PosApplication : Application() {
    private val appScope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    private val _terminalSiap = MutableStateFlow<PlatformInitResult?>(null)
    val terminalSiap: StateFlow<PlatformInitResult?> = _terminalSiap.asStateFlow()

    override fun onCreate() {
        super.onCreate()
        appScope.launch {
            _terminalSiap.value = AiPosAndroid.initPlatform(
                application = this@PosApplication,
                licenseName = "tokoanda.license",   // file di src/main/assets/
            )
        }
    }
}
```

`PlatformInitResult` berisi `success` dan `message`. Tampilkan `message` ke kasir bila
`success` bernilai `false` — misalnya saat file lisensi tidak ditemukan.

### 2. `attachActivity` di `onCreate`

Layar tap kartu diluncurkan lewat `registerForActivityResult`, yang hanya boleh
didaftarkan sebelum Activity mencapai status `STARTED`:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        AiPosAndroid.attachActivity(this)   // sebelum setContent, jangan di onResume
        setContent { PosApp() }
    }
}
```

Melewatkan langkah ini membuat `processPayment()` memancarkan `PaymentState.Failed`
dengan pesan yang menjelaskan penyebabnya.

## Terminal sungguhan vs simulator

Varian terminal ditentukan saat artifact `aipos-payment` **dibangun dan dipublikasikan**:

| Varian | Perilaku |
|---|---|
| **MineSec** | Ponsel NFC menjadi terminal tap-to-pay sungguhan. Butuh lisensi dan `profileId`. |
| **Simulator** | Tidak ada NFC. Setiap transaksi disetujui setelah ±3 detik. Pesan diawali `[SIMULASI]` dan `transactionId` diawali `SIM-`. |

:::danger[Pastikan varian yang Anda terima]
Koordinat Maven yang sama bisa berisi varian mana pun. Sebelum rilis ke kasir, lakukan satu
transaksi uji dan pastikan pesannya **tidak** diawali `[SIMULASI]`. Simulator tidak
memindahkan uang sama sekali.
:::

### Kebutuhan MineSec

1. **Kredensial registry** — lihat [Instalasi](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/installation#repository-minesec-pembayaran-kartu).
2. **File lisensi** (`*.license`) dari MineSec, diletakkan di `app/src/main/assets/`.
   Nama bawaan yang dicari SDK adalah `AiPosAndroid.DEFAULT_LICENSE_NAME`
   (`public-test.license`, lisensi uji).
3. **`profileId`** dari dashboard MineSec, diisi di [`MerchantInfo`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/configuration#identitas-merchant).
4. **Perangkat dengan NFC**. Tambahkan deklarasi berikut bila aplikasi juga harus bisa
   dipasang di perangkat tanpa NFC:

```xml
<uses-feature android:name="android.hardware.nfc" android:required="false" />
```

## Di mana menyimpan instance SDK

Buat **satu instance per proses**. Setiap instance menyimpan keranjang dan riwayat transaksi
di penyimpanan lokal perangkat, dan dua instance yang hidup berbarengan akan saling menimpa.

| Aplikasi Anda | Tempat yang disarankan |
|---|---|
| Satu layar kasir | `ViewModel` layar tersebut |
| Beberapa layar berbagi keranjang | Singleton di `Application` atau container DI Anda (Hilt/Koin) |

SDK memakai container Koin miliknya sendiri, jadi aman dipakai berdampingan dengan Koin atau
Hilt milik aplikasi Anda.

Panggil `close()` saat SDK tidak dipakai lagi. Di `AIPosSDK`, `close()` adalah fungsi
suspend.

## Mengamati state dengan Compose

Semua `observe*()` mengembalikan `Flow`. Ubah jadi `StateFlow` di ViewModel supaya layar
langsung punya nilai awal:

```kotlin
val cart: StateFlow<Cart> = sdk.observeCart()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), Cart())
```

```kotlin
@Composable
fun CartBar(vm: PosViewModel) {
    val cart by vm.cart.collectAsStateWithLifecycle()
    Text("${cart.itemCount} items · ${cart.total.format()}")
}
```

`collectAsStateWithLifecycle` (dari `androidx.lifecycle:lifecycle-runtime-compose`)
menghentikan pengamatan saat aplikasi di latar belakang.

## Memanggil dari Java

Fungsi `suspend` dan `Flow` tidak nyaman dipanggil dari Java. Bungkus di sisi Kotlin, atau
pakai callback yang memang disediakan untuk pembayaran online:

```kotlin
val registration = sdk.addOnlinePaymentListener { status -> /* ... */ }
registration.cancel()   // when the screen is closed
```

## Log

SDK menulis log pembayaran online dengan tag `AiPos/online-payment`:

```bash
adb logcat | grep AiPos/online-payment
```

Saat SDK dirakit, satu baris log mengumumkan mode pembayaran online yang aktif (purwarupa
atau API), alamat backend, dan apakah kunci serta login sudah diatur. Kata sandi tidak
pernah dicetak.

## Contoh lengkap

Aplikasi contoh `sample-android` di repository SDK memperlihatkan integrasi utuh: grid
produk, mikrofon, chip saran AI, keranjang, pembayaran kartu, dan pembayaran online di
bottom sheet.
