# PaymentClient

```kotlin
class PaymentClient
```

Cart and card payment without AI features. Artifact: `com.weekendinc.aipos:aipos-payment`.
Package: `com.weekendinc.aipos.payment`.

Doesn't bring in an OpenAI client or the internet permission.

## Builder

```kotlin
val payment = PaymentClient.Builder()
    .merchantInfo(merchant)
    .productCatalog(catalog)
    .android(context)          // or .ios()
    .build()
```

| Method | Required | Description |
|---|---|---|
| `merchantInfo(info: MerchantInfo)` | yes | Merchant identity. |
| `productCatalog(source: ProductCatalogSource)` | yes | Cart stock is validated against this catalog. |
| `android(context: Context)` | yes (Android) | Extension in `com.weekendinc.aipos.payment`. |
| `ios()` | yes (iOS) | Extension in `com.weekendinc.aipos.payment`. Card payment isn't supported on iOS. |
| `build(): PaymentClient` | — | Throws `IllegalArgumentException` if required configuration is incomplete. |

On Android, [`AiPosAndroid.initPlatform`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/aipos-android) and
`AiPosAndroid.attachActivity` are still required.

## Members

| Member | Type | Description |
|---|---|---|
| `observeCatalog()` | `Flow<List<Product>>` | The catalog contents. |
| `observeCart()` | `Flow<Cart>` | The cart contents. |
| `suspend addToCart(productId: ProductId, quantity: Int = 1)` | `PosResult<Cart>` | Add with stock validation. |
| `suspend updateCartQuantity(productId: ProductId, quantity: Int)` | `PosResult<Cart>` | Quantity ≤ 0 removes the row. |
| `suspend removeFromCart(productId: ProductId)` | `PosResult<Cart>` | Remove the row. |
| `suspend clearCart()` | `Unit` | Clear the cart. |
| `processPayment()` | `Flow<PaymentState>` | Charge the cart total. |
| `suspend cancelPayment()` | `Unit` | Cancel a payment waiting for a tap. |
| `observePaymentState()` | `Flow<PaymentState>` | The current payment status. |
| `observeTransactions(limit: Int = 20)` | `Flow<List<Transaction>>` | Transaction history. |
| `close()` | `Unit` | Release resources. |

See the full guide at [Card payment](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/card-payment).
