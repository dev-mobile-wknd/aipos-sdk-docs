# API Reference

Reference for the entire AI POS SDK version 0.1.1 public API.

## Entry points

| Class | Artifact | Package | Purpose |
|---|---|---|---|
| [`AIPosSDK`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/aipos-sdk) | `aipos-sdk` | `com.weekendinc.aipos` | All features in one object |
| [`AdvisorClient`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/advisor-client) | `aipos-advisor` | `com.weekendinc.aipos.advisor` | Product suggestions only |
| [`PaymentClient`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/payment-client) | `aipos-payment` | `com.weekendinc.aipos.payment` | Cart + card payment |
| [`OnlinePaymentClient`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/online-payment-client) | `aipos-payment-online` | `com.weekendinc.aipos.payment.online` | Online payment |
| [`AiPosAndroid`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/aipos-android) | `aipos-payment` | `com.weekendinc.aipos` | Terminal initialization on Android |

## Configuration and models

| Page | Contents |
|---|---|
| [`OnlinePaymentConfig`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/online-payment-config) | All online payment parameters |
| [Data models](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/models) | `Product`, `Money`, `Cart`, `MerchantInfo`, `PaymentState`, `WebPaymentStatus`, and others |
| [Results & errors](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/results-and-errors) | `PosResult`, `OnlinePaymentErrorCode`, failure messages |
| [Swift bridge](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/swift-bridge) | iOS-specific helper functions |

## Conventions

- **`suspend`** — call it from a coroutine. In Swift it shows up as `async throws`.
- **`Flow<T>`** — a stream of values. In Swift, observe it via
  `FlowSubscriptionKt.subscribe`.
- **`PosResult<T>`** — an operation that can fail doesn't throw an exception; its
  failure is returned as `PosResult.Failure` with a message that's safe to
  display.
- **`IllegalArgumentException`** is only thrown by `Builder.build()` when required
  configuration is missing.

## Common imports

```kotlin
import com.weekendinc.aipos.AIPosSDK
import com.weekendinc.aipos.AiPosAndroid
import com.weekendinc.aipos.android                         // Builder.android(context) extension
import com.weekendinc.aipos.domain.catalog.MutableProductCatalog
import com.weekendinc.aipos.domain.catalog.ProductCatalogSource
import com.weekendinc.aipos.domain.common.PosResult
import com.weekendinc.aipos.domain.entity.Cart
import com.weekendinc.aipos.domain.entity.MerchantInfo
import com.weekendinc.aipos.domain.entity.Product
import com.weekendinc.aipos.domain.entity.WebPaymentStatus
import com.weekendinc.aipos.domain.model.PaymentState
import com.weekendinc.aipos.domain.valueobject.Money
import com.weekendinc.aipos.domain.valueobject.ProductId
import com.weekendinc.aipos.payment.online.AiposPaymentWebView
import com.weekendinc.aipos.payment.online.AiposPaymentWebViewClient
import com.weekendinc.aipos.payment.online.OnlinePaymentConfig
import com.weekendinc.aipos.payment.online.paymentPageProbe
```

:::warning[Internal API]
Declarations marked `@InternalAiPosApi` are used between SDK modules and aren't
part of the public API. The compiler refuses to let your app use them, and their
shape can change without notice.
:::
