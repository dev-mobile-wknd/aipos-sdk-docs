# AIPosSDK

```kotlin
class AIPosSDK : WebPaymentUrlListener
```

The single entry point for all features. Artifact: `com.weekendinc.aipos:aipos-sdk`.

## Builder

```kotlin
val sdk = AIPosSDK.Builder()
    .llmApiKey(key)                 // or .llmProxyBaseUrl(url)
    .merchantInfo(merchant)
    .productCatalog(catalog)
    .onlinePayment(config)          // optional
    .android(context)               // or .ios()
    .build()
```

| Method | Required | Description |
|---|---|---|
| `llmApiKey(key: String)` | one of | Direct OpenAI key. For development. If a proxy is also set, this is sent as a token to the proxy. |
| `llmProxyBaseUrl(url: String)` | one of | An OpenAI-compatible endpoint, **without** `/v1`. Takes priority if both are set. |
| `merchantInfo(info: MerchantInfo)` | yes | Merchant identity. |
| `productCatalog(source: ProductCatalogSource)` | yes | Product catalog source. |
| `onlinePayment(config: OnlinePaymentConfig)` | no | Default: `OnlinePaymentConfig()` (beta environment). |
| `android(context: Context)` | yes (Android) | Extension in `com.weekendinc.aipos`. |
| `ios()` | yes (iOS) | Extension in `com.weekendinc.aipos`. |
| `build(): AIPosSDK` | — | Throws `IllegalArgumentException` if required configuration is incomplete. |

## Catalog

| Member | Type | Description |
|---|---|---|
| `observeCatalog()` | `Flow<List<Product>>` | The catalog contents from `productCatalog(...)`. |

## Product suggestions

| Member | Type | Description |
|---|---|---|
| `pushTranscript(text: String)` | `Unit` | Submit one final sentence. Safe to call as often as you like. |
| `setInterimTranscript(text: String)` | `Unit` | A partial sentence still being recognized. |
| `requestSuggestions()` | `Unit` | Analyze now, without waiting for the conversation to settle. |
| `clearAdvisor()` | `Unit` | Clear the transcript and suggestions. The cart isn't touched. |
| `observeSuggestions()` | `Flow<List<ProductSuggestion>>` | Latest suggestions, sorted from most relevant. |
| `observeBuyerIntent()` | `Flow<String>` | A summary of the customer's needs. |
| `observeAdvisorState()` | `Flow<AdvisorState>` | `Idle`, `Thinking`, `Suggesting`. |
| `observeTranscript()` | `Flow<List<String>>` | Every sentence submitted so far. |
| `observeInterimTranscript()` | `Flow<String>` | The last partial sentence. |
| `observeAdvisorError()` | `Flow<String?>` | The latest failure; `null` if none. |

## Cashier agent

| Member | Type | Description |
|---|---|---|
| `suspend sendMessage(message: String)` | `String` | Send a command; returns once the agent is done. |
| `observeMessages()` | `Flow<List<AgentMessage>>` | The conversation history. |
| `observeProcessing()` | `Flow<Boolean>` | `true` while the agent is processing a message. |

## Cart

| Member | Type | Description |
|---|---|---|
| `observeCart()` | `Flow<Cart>` | The cart contents. |
| `suspend addToCart(productId: ProductId, quantity: Int = 1)` | `PosResult<Cart>` | Add with stock validation. |
| `suspend updateCartQuantity(productId: ProductId, quantity: Int)` | `PosResult<Cart>` | Quantity ≤ 0 removes the row. |
| `suspend removeFromCart(productId: ProductId)` | `PosResult<Cart>` | Remove the row. |
| `suspend clearCart()` | `Unit` | Clear the cart. |

## Card payment

| Member | Type | Description |
|---|---|---|
| `processPayment()` | `Flow<PaymentState>` | Charge the cart total. The stream ends at `Success` or `Failed`. |
| `suspend cancelPayment()` | `Unit` | Cancel a payment waiting for a tap. |
| `observePaymentState()` | `Flow<PaymentState>` | The current status, including payments run by the agent. |
| `observeTransactions(limit: Int = 20)` | `Flow<List<Transaction>>` | Transaction history on the device. |

## Online payment

| Member | Type | Description |
|---|---|---|
| `onlinePaymentStatus` | `StateFlow<WebPaymentStatus>` | The status of the running session; `Idle` if there isn't one. |
| `suspend startOnlinePayment(note: String? = null)` | `PosResult<WebPaymentSession>` | Open a session for the cart contents. |
| `attachPaymentPageProbe(probe: PaymentPageProbe)` | `Unit` | **Required.** Attach the WebView loading the payment page. |
| `onWebViewUrlChanged(url: String)` | `Unit` | Report the WebView's URL. Already done by `AiposPaymentWebViewClient`. |
| `onWebViewLoadFailed(reason: String)` | `Unit` | Report that the page failed to load. |
| `addOnlinePaymentListener(onStatus: PaymentStatusListener)` | `PaymentListenerRegistration` | A callback alternative to `Flow`. Cancel with `cancel()`. |
| `cancelOnlinePayment()` | `Unit` | Cancel the session. The cart isn't cleared. |

## Lifecycle

| Member | Type | Description |
|---|---|---|
| `suspend resetSession()` | `Unit` | Clear the conversation, suggestions, cart, payment status, and online session. |
| `suspend close()` | `Unit` | Release all resources. The instance must not be used afterward. |
