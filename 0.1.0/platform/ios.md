# iOS integration

On iOS, every feature works **except contactless card payment**. Product suggestions,
the cashier agent, the cart, and online payment work fully.

## Choosing an integration path

A Maven artifact can only be consumed by Gradle. The path you take depends on the
shape of your iOS app:

| Your iOS app | Path |
|---|---|
| Already uses **Kotlin Multiplatform** (has a `shared` module) | [A. Through the shared module](#a-through-the-shared-module-kotlin-multiplatform) |
| **Pure Swift** without Gradle | [B. Swift Package Manager](#b-swift-package-manager-for-swift-apps) |

### A. Through the shared module (Kotlin Multiplatform)

Add the SDK as an `api` dependency and **export** it to your iOS framework. Without
`export`, the SDK's types aren't visible from Swift.

```kotlin title="shared/build.gradle.kts"
kotlin {
    listOf(iosArm64(), iosSimulatorArm64(), iosX64()).forEach { target ->
        target.binaries.framework {
            baseName = "Shared"
            isStatic = true
            export("com.weekendinc.aipos:aipos-sdk:0.1.0")
            export("com.weekendinc.aipos:aipos-core:0.1.0")
            export("com.weekendinc.aipos:aipos-advisor:0.1.0")
            export("com.weekendinc.aipos:aipos-agent:0.1.0")
            export("com.weekendinc.aipos:aipos-payment:0.1.0")
            export("com.weekendinc.aipos:aipos-payment-online:0.1.0")
        }
    }

    sourceSets {
        commonMain.dependencies {
            api("com.weekendinc.aipos:aipos-sdk:0.1.0")
        }
    }
}
```

In Swift, `import Shared` and then use the examples below. Kotlin 2.3.21 or newer is
required for the SDK's klib to be readable.

### B. Swift Package Manager for Swift apps

A pure Swift app adds `AIPosSDK` via Swift Package Manager — not via Maven. Its
binary (`AIPosSDK.xcframework`, static) is hosted in a public distribution repo
separate from the SDK source:

1. Xcode → **File → Add Package Dependencies…**
2. Enter the URL: `https://github.com/dev-mobile-wknd/swift-aipos-sdk`
3. Choose the version you want (following release tags, e.g. `0.1.0`)
4. Xcode downloads the binary, verifies the checksum, and links it automatically

After the package is added, set one extra build setting on the app target:

```
OTHER_LDFLAGS = $(inherited) -lc++
```

:::warning[`-lc++` is required]
The Kotlin/Native runtime uses libc++, and it isn't carried along in a static
framework — SwiftPM doesn't add it automatically for a binary target. Without
`-lc++`, linking fails with `std::` symbols not found.
:::

:::info[Why a separate, public distribution repo?]
That repo only contains `Package.swift` and the compiled binary — the Kotlin SDK
source stays private. It's public because SwiftPM can't authenticate `binaryTarget`
downloads from a private GitHub Release. This doesn't leak anything sensitive: the
SDK is still useless without an `llmApiKey`, `merchantInfo`, and payment
credentials issued through the merchant onboarding process.
:::

## Assembling the SDK from Swift

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

:::info[Default values don't cross over to Swift]
Kotlin parameters that have a default value are still required from Swift. That's
why `MerchantInfo` above writes `profileId` explicitly, and `Product` is built via
`productOf` instead of its constructor.
:::

## Observing a `Flow` from Swift

A `collect` function called from Swift **doesn't get cancelled** when the Swift
`Task` is cancelled. Use `FlowSubscriptionKt.subscribe`, which returns a handle to
stop it:

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

The value received is typed `Any?` because Kotlin generics don't carry over to
Objective-C — downcast it yourself with `as?`.

## Calling a `suspend` function

Suspend functions show up as `async throws` functions in Swift:

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

`PosResult` reads as `PosResultSuccess<T>` and `PosResultFailure`.

## What changes shape in Swift

| Kotlin | In Swift | Workaround |
|---|---|---|
| `Money` (price, total) | `Int64` in **cents** | `IosDomainFactoryKt.formatMoneyCents(cents:)` → `"Rp 13.999.000"` |
| `ProductId` | `Any` | Pass `product.id` through as-is; read its text with `IosDomainFactoryKt.productIdValue(product:)` |
| `Product` constructor | Unusable | `IosDomainFactoryKt.productOf(...)` with the price in whole Rupiah |
| `OnlinePaymentConfig` constructor | 15 required arguments | `IosOnlinePaymentConfigFactoryKt.onlinePaymentConfigOf(...)` |
| `Flow<T>` | Can't `for await` | `FlowSubscriptionKt.subscribe(flow) { }` |
| `AiposPaymentNavigationDelegate` | `unavailable` | Write your own `WKNavigationDelegate` — see below |

Details on all the bridge functions are in the [Swift bridge](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/swift-bridge).

## Online payment in `WKWebView`

Online payment needs three connections: a navigation delegate, KVO on `url`, and a
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

Reporting the same URL repeatedly has no effect, so KVO and the delegate are safe
to use together. The full flow and emitted statuses are explained in
[Online payment](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/online-payment).

## Permissions in `Info.plist`

If you use the microphone for product suggestions, both of these keys are
required. Missing either one causes the app to be force-closed when the
microphone is turned on.

```xml
<key>NSMicrophoneUsageDescription</key>
<string>The microphone is used to listen to conversations with customers.</string>
<key>NSSpeechRecognitionUsageDescription</key>
<string>Conversations are converted to text so the AI can suggest products.</string>
```

## Card payment on iOS

`processPayment()` on iOS immediately emits a single `PaymentState.Failed`
explaining that contactless payment isn't supported. Hide the card payment button
on iOS, and offer [online payment](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/online-payment) instead.
