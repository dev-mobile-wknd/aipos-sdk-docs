# Data models

Every model is an *immutable* `data class` or `sealed class`.

## Base values

### `Money`

Package `com.weekendinc.aipos.domain.valueobject`. Money is stored as an integer
in **cents** (1/100 Rupiah) to avoid rounding errors.

| Member | Description |
|---|---|
| `Money.fromRupiah(amount: Long)` | Create from whole Rupiah. `fromRupiah(25_000)` = Rp 25,000. |
| `Money(cents: Long)` | Create from cents. `Money(2_500_000)` = Rp 25,000. |
| `Money.ZERO` | Zero. |
| `cents` | The value in cents. |
| `whole` | The value in whole Rupiah. |
| `format()` | `"Rp 25.000"`. Also used by `toString()`. |
| `isNotPositive` | `true` when zero or negative. |
| `+`, `-`, `* Int`, `compareTo` | Arithmetic and comparison operations. |

### `ProductId`

Package `com.weekendinc.aipos.domain.valueobject`. Wraps a `String`; may not be
empty.

```kotlin
val id = ProductId("GDG-001")
id.value   // "GDG-001"
```

## Catalog and cart

Package `com.weekendinc.aipos.domain.entity`.

### `Product`

| Field | Type | Default |
|---|---|---|
| `id` | `ProductId` | — |
| `name` | `String` | — |
| `price` | `Money` | — |
| `barcode` | `String` | — |
| `category` | `String` | — |
| `stock` | `Int` | — |
| `description` | `String` | `""` |
| `imageUrl` | `String` | `""` |
| `isAvailable` | `Boolean` (derived) | `stock > 0` |

### `Cart`

| Member | Type |
|---|---|
| `items` | `List<CartItem>` |
| `discount` | `Money` |
| `subtotal` | `Money` |
| `total` | `Money` — never negative |
| `itemCount` | `Int` — total quantity |
| `isEmpty` | `Boolean` |
| `findItem(productId)` | `CartItem?` |

### `CartItem`

`product: Product`, `quantity: Int` (at least 1), `subtotal: Money`.

### `MerchantInfo`

| Field | Type | Default |
|---|---|---|
| `id` | `String` | — |
| `name` | `String` | — |
| `address` | `String` | — |
| `terminalId` | `String` | — |
| `mid` | `String` | — |
| `profileId` | `String` | `MerchantInfo.DEFAULT_PROFILE_ID` (test profile) |

## Product suggestions

Package `com.weekendinc.aipos.advisor`.

### `ProductSuggestion`

| Field | Type | Description |
|---|---|---|
| `product` | `Product` | The full product from the catalog |
| `reason` | `String` | The reasoning, in Indonesian |
| `confidence` | `Double` | 0.0 – 1.0 |

### `AdvisorState`

A `sealed class` with three objects: `Idle`, `Thinking`, `Suggesting`.

## Cashier agent

### `AgentMessage`

Package `com.weekendinc.aipos.agent.model`. Every subtype has a
`timestamp: Long`.

| Subtype | Field |
|---|---|
| `UserMessage` | `content: String` |
| `AssistantMessage` | `content: String` |
| `ToolCall` | `toolName: String`, `summary: String` |
| `ErrorMessage` | `message: String` |

## Card payment

### `PaymentState`

Package `com.weekendinc.aipos.domain.model`.

| Subtype | Field |
|---|---|
| `Idle` | — |
| `WaitingTap` | `message: String`, `timeoutSeconds: Int` |
| `Processing` | `message: String`, `cardScheme: CardScheme` |
| `Success` | `transactionId: String`, `amount: Money`, `paymentMethod: PaymentMethod`, `cardScheme: CardScheme`, `rrn: String?`, `approvalCode: String?`, `posReference: String?` |
| `Failed` | `errorMessage: String`, `responseCode: String?`, `isCancelled: Boolean`, `isTimeout: Boolean` |

`isTerminal: Boolean` — `true` for `Success` and `Failed`.

### `Transaction`

Package `com.weekendinc.aipos.domain.entity`.

| Field | Type |
|---|---|
| `id` | `String` |
| `cart` | `Cart` |
| `amount` | `Money` |
| `paymentMethod` | `PaymentMethod` |
| `status` | `TransactionStatus` |
| `timestamp` | `Long` |
| `rrn` | `String?` |
| `approvalCode` | `String?` |
| `posReference` | `String?` |
| `isApproved` | `Boolean` (derived) |

### `TransactionStatus`

`PENDING`, `APPROVED`, `DECLINED`, `CANCELLED`, `TIMEOUT`.

### `PaymentMethod`

| Subtype | Field | `displayName` |
|---|---|---|
| `Contactless` | `scheme: CardScheme`, `maskedPan: String?` | `"VISA **** 4242"` |
| `QRCode` | `issuer: String = "QRIS"` | `"QRIS"` |
| `Cash` | — | `"TUNAI"` (Indonesian for "cash") |

### `CardScheme`

`VISA`, `MASTERCARD`, `AMEX`, `JCB`, `MAESTRO`, `UNIONPAY`, `UNKNOWN`.

## Online payment

Package `com.weekendinc.aipos.domain.entity`.

### `WebPaymentSession`

| Field | Type |
|---|---|
| `sessionId` | `String` |
| `orderId` | `String` |
| `paymentUrl` | `String` |
| `amount` | `Money` |
| `expiresAtMillis` | `Long?` |

### `WebPaymentStatus`

Every subtype has a `transactionId: String?`.

| Subtype | Extra field | `isFinal` |
|---|---|:---:|
| `Idle` | — | |
| `ChoosingMethod` | — | |
| `Pending` | — | |
| `Success` | — | ✅ |
| `Failed` | `rawStatus: String?` | ✅ |
| `Cancelled` | — | ✅ |
| `Unknown` | `rawStatus: String` | |

### `OnlinePaymentChannel`

| Value | `code` |
|---|---|
| `VIRTUAL_ACCOUNT` | `"VA"` |
| `QRIS` | `"QR"` |
| `CARD` | `"CARD"` |

`OnlinePaymentChannel.ALL` and `OnlinePaymentChannel.DEFAULT` — both contain all
three channels as of version 0.1.0.
