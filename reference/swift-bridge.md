# Swift bridge

Some Kotlin APIs don't carry over to Swift intact. The SDK provides helper
functions in its iOS source set to close those gaps. Swift class names follow
their Kotlin file name (`FileNameKt`).

## `FlowSubscriptionKt.subscribe`

```swift
func subscribe(_ flow: Kotlinx_coroutines_coreFlow, onEach: @escaping (Any?) -> Void) -> FlowSubscription
```

Observe a `Flow` from Swift. Every value is delivered **on the main thread**.

```swift
let subscription = FlowSubscriptionKt.subscribe(sdk.observeCart()) { value in
    guard let cart = value as? Cart else { return }
    self.cart = cart
}
subscription.cancel()   // required when the screen is closed
```

`FlowSubscription` has `isActive: Bool` and `cancel()` (safe to call more than
once).

## `IosDomainFactoryKt`

### `productOf`

```swift
IosDomainFactoryKt.productOf(
    id: String, name: String, priceRupiah: Int64,
    barcode: String, category: String, stock: Int32,
    description: String, imageUrl: String
) -> Product
```

Builds a `Product` from plain types. The price is in **whole Rupiah**.

### `formatMoneyCents`

```swift
IosDomainFactoryKt.formatMoneyCents(cents: Int64) -> String
```

Formats a money value from properties like `cart.total` or `product.price`
(typed `Int64` in cents) into `"Rp 13.999.000"`.

### `productIdValue`

```swift
IosDomainFactoryKt.productIdValue(product: Product) -> String
```

Reads a product's id as a `String` — for example, for `Identifiable` in
SwiftUI. To call `addToCart(productId:)`, pass `product.id` through as-is.

## `IosOnlinePaymentConfigFactoryKt`

### `onlinePaymentConfigOf`

```swift
// Device key only; everything else uses the SDK default (beta environment)
IosOnlinePaymentConfigFactoryKt.onlinePaymentConfigOf(apiKey: String) -> OnlinePaymentConfig

// Full
IosOnlinePaymentConfigFactoryKt.onlinePaymentConfigOf(
    baseUrl: String, apiKey: String, email: String, password: String,
    loopbackUrlReplacement: String
) -> OnlinePaymentConfig
```

Other parameters (`linkExpiry`, `appearance`, and so on) use their default
values. To change them, assemble `OnlinePaymentConfig` on the Kotlin side.

## `IosPaymentPageProbeKt`

### `paymentPageProbe`

```swift
IosPaymentPageProbeKt.paymentPageProbe(_ webView: WKWebView) -> PaymentPageProbe
```

The probe for `sdk.attachPaymentPageProbe(probe:)`. Required for the online
payment status to move.

## Types that change shape

| Kotlin | Swift |
|---|---|
| `Money` | `Int64` (cents) |
| `ProductId` | `Any` |
| `Flow<T>` | `Kotlinx_coroutines_coreFlow` — value is `Any?` |
| `PosResult.Success<T>` | `PosResultSuccess<T>` — data in `.data` |
| `PosResult.Failure` | `PosResultFailure` — message in `.message` |
| A `sealed class` object (`AdvisorState.Idle`) | A class initialized as `AdvisorState.Idle()`; compare with `is` |
| A companion constant (`MerchantInfo.DEFAULT_PROFILE_ID`) | `MerchantInfo.companion.DEFAULT_PROFILE_ID` |
| A `suspend` function | `async throws` |
| A parameter with a default value | Required |
