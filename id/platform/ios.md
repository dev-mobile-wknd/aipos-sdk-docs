# Integrasi iOS

Di iOS, semua fitur berjalan **kecuali pembayaran kartu contactless**. Penyaran produk,
agent kasir, keranjang, dan pembayaran online berfungsi penuh.

## Memilih jalur integrasi

Artifact Maven hanya bisa dikonsumsi oleh Gradle. Jalur yang Anda tempuh bergantung pada
bentuk aplikasi iOS Anda:

| Aplikasi iOS Anda | Jalur |
|---|---|
| Sudah memakai **Kotlin Multiplatform** (ada modul `shared`) | [A. Lewat modul shared](#a-lewat-modul-shared-kotlin-multiplatform) |
| **Swift murni** tanpa Gradle | [B. Swift Package Manager](#b-swift-package-manager-untuk-aplikasi-swift) |

### A. Lewat modul shared (Kotlin Multiplatform)

Tambahkan SDK sebagai dependensi `api` dan **ekspor** ke framework iOS Anda. Tanpa `export`,
tipe milik SDK tidak terlihat dari Swift.

```kotlin title="shared/build.gradle.kts"
kotlin {
    listOf(iosArm64(), iosSimulatorArm64(), iosX64()).forEach { target ->
        target.binaries.framework {
            baseName = "Shared"
            isStatic = true
            export("com.weekendinc.aipos:aipos-sdk:0.1.1")
            export("com.weekendinc.aipos:aipos-core:0.1.1")
            export("com.weekendinc.aipos:aipos-advisor:0.1.1")
            export("com.weekendinc.aipos:aipos-agent:0.1.1")
            export("com.weekendinc.aipos:aipos-payment:0.1.1")
            export("com.weekendinc.aipos:aipos-payment-online:0.1.1")
        }
    }

    sourceSets {
        commonMain.dependencies {
            api("com.weekendinc.aipos:aipos-sdk:0.1.1")
        }
    }
}
```

Di Swift, `import Shared` lalu pakai contoh-contoh di bawah. Kotlin versi 2.3.21 atau lebih
baru diperlukan agar klib SDK bisa dibaca.

### B. Swift Package Manager untuk aplikasi Swift

Aplikasi Swift murni menambahkan `AIPosSDK` lewat Swift Package Manager — bukan lewat Maven.
Binary-nya (`AIPosSDK.xcframework`, static) di-host di repo distribusi publik terpisah dari
source SDK:

1. Xcode → **File → Add Package Dependencies…**
2. Masukkan URL: `https://github.com/dev-mobile-wknd/swift-aipos-sdk`
3. Pilih versi yang diinginkan (mengikuti tag rilis, misalnya `0.1.1`)
4. Xcode mengunduh binary-nya, memverifikasi checksum, dan menautkannya secara otomatis

Setelah package ditambahkan, atur satu setelan build tambahan di target aplikasi:

```
OTHER_LDFLAGS = $(inherited) -lc++
```

:::warning[`-lc++` wajib]
Runtime Kotlin/Native memakai libc++, dan pada framework statis pustaka itu tidak ikut
terbawa — SwiftPM tidak menambahkannya otomatis untuk binary target. Tanpa `-lc++`,
penautan gagal dengan simbol `std::` yang tidak ditemukan.
:::

:::info[Kenapa repo distribusi terpisah dan publik?]
Repo itu hanya berisi `Package.swift` dan binary hasil kompilasi — source Kotlin SDK tetap
privat. Dibuat publik karena SwiftPM tidak bisa mengautentikasi unduhan `binaryTarget` dari
GitHub Release privat. Ini tidak membocorkan apa pun yang sensitif: SDK tetap tidak berguna
tanpa `llmApiKey`, `merchantInfo`, dan kredensial payment yang diterbitkan lewat proses
onboarding merchant.
:::

## Merakit SDK dari Swift

```swift
import AIPosSDK

let merchant = MerchantInfo(
    id: "merchant_001",
    name: "Jaya Gadget Store",
    address: "Jl. Sudirman No. 1, Jakarta",
    terminalId: "TID001",
    mid: "MID123456",
    profileId: MerchantInfo.companion.DEFAULT_PROFILE_ID
)

let catalog = MutableProductCatalog(initial: [
    IosDomainFactoryKt.productOf(
        id: "GDG-001", name: "iPhone 15 128GB", priceRupiah: 13_999_000,
        barcode: "8991234500011", category: "Smartphone", stock: 12,
        description: "48MP camera, USB-C", imageUrl: ""
    ),
])

let sdk = AIPosSDK.Builder()
    .llmApiKey(key: apiKey)
    .merchantInfo(info: merchant)
    .productCatalog(source: catalog)
    .onlinePayment(config: IosOnlinePaymentConfigFactoryKt.onlinePaymentConfigOf(apiKey: posApiKey))
    .ios()
    .build()
```

:::info[Nilai bawaan tidak menyeberang ke Swift]
Parameter Kotlin yang punya nilai bawaan tetap wajib diisi dari Swift. Karena itu
`MerchantInfo` di atas menuliskan `profileId` secara eksplisit, dan `Product` dirakit lewat
`productOf` alih-alih konstruktornya.
:::

## Mengamati `Flow` dari Swift

Fungsi `collect` yang dipanggil dari Swift **tidak ikut batal** ketika `Task` Swift
dibatalkan. Pakai `FlowSubscriptionKt.subscribe`, yang mengembalikan pegangan untuk
menghentikannya:

```swift
@MainActor
final class PosViewModel: ObservableObject {
    @Published var cart: Cart?
    private var subscriptions: [FlowSubscription] = []

    func start() {
        subscriptions.append(FlowSubscriptionKt.subscribe(sdk.observeCart()) { [weak self] value in
            guard let cart = value as? Cart else { return }
            self?.cart = cart                   // already on the main thread
        })
    }

    func stop() {
        subscriptions.forEach { $0.cancel() }
        subscriptions.removeAll()
    }
}
```

Nilai yang diterima bertipe `Any?` karena generik Kotlin tidak terbawa ke Objective-C —
turunkan sendiri dengan `as?`.

## Memanggil fungsi `suspend`

Fungsi suspend muncul sebagai fungsi `async throws` di Swift:

```swift
Task {
    do {
        let result = try await sdk.addToCart(productId: product.id, quantity: 1)
        if let failure = result as? PosResultFailure {
            errorMessage = failure.message
        }
    } catch {
        errorMessage = error.localizedDescription
    }
}
```

`PosResult` terbaca sebagai `PosResultSuccess<T>` dan `PosResultFailure`.

## Yang berubah bentuk di Swift

| Kotlin | Di Swift | Solusinya |
|---|---|---|
| `Money` (harga, total) | `Int64` berisi **sen** | `IosDomainFactoryKt.formatMoneyCents(cents:)` → `"Rp 13.999.000"` |
| `ProductId` | `Any` | Oper `product.id` apa adanya; baca teksnya dengan `IosDomainFactoryKt.productIdValue(product:)` |
| Konstruktor `Product` | Tidak bisa dipakai | `IosDomainFactoryKt.productOf(...)` dengan harga dalam Rupiah penuh |
| Konstruktor `OnlinePaymentConfig` | 15 argumen wajib | `IosOnlinePaymentConfigFactoryKt.onlinePaymentConfigOf(...)` |
| `Flow<T>` | Tidak bisa di-`for await` | `FlowSubscriptionKt.subscribe(flow) { }` |
| `AiposPaymentNavigationDelegate` | `unavailable` | Tulis `WKNavigationDelegate` sendiri — lihat di bawah |

Rincian semua fungsi jembatan ada di [Jembatan Swift](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/reference/swift-bridge).

## Pembayaran online di `WKWebView`

Pembayaran online membutuhkan tiga sambungan: delegate navigasi, KVO pada `url`, dan
*page probe*.

```swift
import WebKit

final class PaymentWebViewController: UIViewController, WKNavigationDelegate {
    private let sdk: AIPosSDK
    private let session: WebPaymentSession
    private var webView: WKWebView!
    private var urlObservation: NSKeyValueObservation?

    init(sdk: AIPosSDK, session: WebPaymentSession) {
        self.sdk = sdk
        self.session = session
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        webView = WKWebView(frame: view.bounds)
        webView.navigationDelegate = self
        view.addSubview(webView)

        // 1. Single-page navigation happens via history.pushState, which does
        //    not call the delegate. KVO catches it.
        urlObservation = webView.observe(\.url) { [weak self] view, _ in
            if let url = view.url?.absoluteString { self?.sdk.onWebViewUrlChanged(url: url) }
        }

        // 2. REQUIRED: without the probe, the status gets stuck at "choosing a method".
        sdk.attachPaymentPageProbe(probe: IosPaymentPageProbeKt.paymentPageProbe(webView))

        webView.load(URLRequest(url: URL(string: session.paymentUrl)!))
    }

    // 3. Navigation delegate
    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        if let url = webView.url?.absoluteString { sdk.onWebViewUrlChanged(url: url) }
    }

    func webView(_ webView: WKWebView, didFailProvisionalNavigation navigation: WKNavigation!, withError error: Error) {
        sdk.onWebViewLoadFailed(reason: error.localizedDescription)
    }
}
```

Melaporkan URL yang sama berkali-kali tidak berdampak apa pun, jadi KVO dan delegate aman
dipakai bersamaan. Alur lengkap dan status yang dipancarkan dijelaskan di
[Pembayaran online](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/online-payment).

## Izin di `Info.plist`

Bila Anda memakai mikrofon untuk penyaran produk, kedua kunci ini wajib ada. Menghilangkan
salah satunya membuat aplikasi dihentikan paksa saat mikrofon dinyalakan.

```xml
<key>NSMicrophoneUsageDescription</key>
<string>The microphone is used to listen to conversations with customers.</string>
<key>NSSpeechRecognitionUsageDescription</key>
<string>Conversations are converted to text so the AI can suggest products.</string>
```

## Pembayaran kartu di iOS

`processPayment()` di iOS langsung memancarkan satu `PaymentState.Failed` yang menjelaskan
bahwa pembayaran contactless belum didukung. Sembunyikan tombol bayar kartu di iOS, dan
tawarkan [pembayaran online](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/online-payment) sebagai gantinya.
