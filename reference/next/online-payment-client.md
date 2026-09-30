# OnlinePaymentClient

```kotlin
class OnlinePaymentClient : WebPaymentUrlListener
```

Cart and online payment without a terminal and without AI features. Artifact:
`com.weekendinc.aipos:aipos-payment-online`. Package: `com.weekendinc.aipos.payment.online`.

## Builder

```kotlin
val online = OnlinePaymentClient.Builder()
    .merchantInfo(merchant)
    .productCatalog(catalog)
    .config(OnlinePaymentConfig(apiKey = key))   // optional
    .android(context)                             // or .ios()
    .build()
```

| Method | Required | Description |
|---|---|---|
| `merchantInfo(info: MerchantInfo)` | yes | Merchant identity. |
| `productCatalog(source: ProductCatalogSource)` | yes | The price and stock being charged. |
| `config(config: OnlinePaymentConfig)` | no | Default: `OnlinePaymentConfig()` (beta environment). |
| `android(context: Context)` | yes (Android) | Extension in `com.weekendinc.aipos.payment.online`. |
| `ios()` | yes (iOS) | Extension in `com.weekendinc.aipos.payment.online`. |
| `build(): OnlinePaymentClient` | — | Throws `IllegalArgumentException` if required configuration is incomplete. |

## Members

| Member | Equivalent on `AIPosSDK` | Type |
|---|---|---|
| `observeCatalog()` | same | `Flow<List<Product>>` |
| `observeCart()` | same | `Flow<Cart>` |
| `suspend addToCart(productId: ProductId, quantity: Int = 1)` | same | `PosResult<Cart>` — stock-validated |
| `suspend updateCartQuantity(productId: ProductId, quantity: Int)` | same | `PosResult<Cart>` — quantity ≤ 0 removes the row |
| `suspend removeFromCart(productId: ProductId)` | same | `PosResult<Cart>` |
| `suspend clearCart()` | same | `Unit` |
| `status` | `onlinePaymentStatus` | `StateFlow<WebPaymentStatus>` |
| `suspend startSession(note: String? = null)` | `startOnlinePayment(note)` | `PosResult<WebPaymentSession>` |
| `attachPaymentPageProbe(probe: PaymentPageProbe)` | same | `Unit` |
| `onWebViewUrlChanged(url: String)` | same | `Unit` |
| `onWebViewLoadFailed(reason: String)` | same | `Unit` |
| `addStatusListener(onStatus: PaymentStatusListener)` | `addOnlinePaymentListener(onStatus)` | `PaymentListenerRegistration` |
| `cancelSession()` | `cancelOnlinePayment()` | `Unit` |
| `resetSession()` | — | `Unit` — returns to `Idle`; the cart isn't touched |
| `close()` | `close()` | `Unit` |

## WebView adapters

| Declaration | Platform | Description |
|---|---|---|
| `AiposPaymentWebView(context)` | Android | A pre-configured `WebView` that stays scrollable inside a bottom sheet |
| `AiposPaymentWebViewClient(listener: WebPaymentUrlListener)` | Android | Forwards URL changes and load failures to the SDK |
| `WebView.prepareForAiposPayment()` | Android | Turns on JavaScript, DOM storage, and viewport adjustments on a plain `WebView` |
| `WebView.paymentPageProbe(): PaymentPageProbe` | Android | The probe for `attachPaymentPageProbe` |
| `WKWebView.paymentPageProbe(): PaymentPageProbe` | iOS | In Swift: `IosPaymentPageProbeKt.paymentPageProbe(webView)` |
| `AiposPaymentNavigationDelegate(listener)` | iOS (Kotlin only) | Can't be used from Swift — write your own `WKNavigationDelegate` |

See the full guide at [Online payment](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/online-payment).
