# OnlinePaymentConfig

```kotlin
data class OnlinePaymentConfig(...)
```

Package: `com.weekendinc.aipos.payment.online`. Passed to
`AIPosSDK.Builder.onlinePayment(...)` or `OnlinePaymentClient.Builder.config(...)`.

Every parameter has a default value. Always use **named arguments**.

## Connection and authentication

| Parameter | Type | Default | Description |
|---|---|---|---|
| `baseUrl` | `String` | `DEFAULT_BASE_URL` (beta backend) | The API address without a trailing `/` and without `/v1`. **An empty string = prototype mode** (no HTTP). |
| `apiKey` | `String` | `""` | The device key, sent as the `X-API-Key` header. The only one without a meaningful default. |
| `email` | `String` | `DEFAULT_EMAIL` (beta account) | The account exchanged for a token via `POST {baseUrl}/v1/login`. |
| `password` | `String` | `DEFAULT_PASSWORD` (beta account) | Paired with `email`. Ends up bundled in the app — see [Production checklist](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guides/production-checklist#key-security). |
| `requestTimeoutMillis` | `Long` | `10_000` | The timeout for a single HTTP request. |
| `loopbackUrlReplacement` | `String` | `DEFAULT_LOOPBACK_URL_REPLACEMENT` (prototype page) | The scheme+host used to replace a `localhost` address the API returns. **Set to `""` in production.** |

## Billing

| Parameter | Type | Default | Description |
|---|---|---|---|
| `linkExpiry` | `kotlin.time.Duration` | `24.hours` | How long the payment link lives from creation. |
| `adminFee` | `Money` | `Money.ZERO` | An admin fee on top of the item price. Zero means it isn't sent. |
| `allowedPaymentMethods` | `Set<OnlinePaymentChannel>` | `OnlinePaymentChannel.DEFAULT` (VA, QRIS, card) | The channels the customer may choose, shown in that order. |

## Page appearance

| Parameter | Type | Default | Description |
|---|---|---|---|
| `appearance` | `PaymentLinkAppearance` | empty | The page's logo and colors. Empty fields aren't sent. |
| `customerData` | `CustomerDataPolicy` | all off | Customer data requested on the page. |
| `hiddenButtonLabels` | `List<String>` | `["back to merchant", "kembali ke merchant"]` | Buttons/links whose text **contains** one of these (case-insensitive) are hidden. An empty list turns the feature off. |

### `PaymentLinkAppearance`

| Field | Type | Example |
|---|---|---|
| `logoUrl` | `String` | `"https://tokoanda.co.id/logo.png"` — must be publicly reachable |
| `backgroundColor` | `String` | `"#9e086c"` |
| `buttonColor` | `String` | `"#273395"` |
| `textMode` | `PaymentLinkTextMode?` | `LIGHT`, `DARK`, or `null` (provider default) |

### `CustomerDataPolicy`

Contains `name`, `email`, `phone`, and `address`, each typed
`CollectCustomerField(enabled: Boolean = false, required: Boolean = false)`.

```kotlin
customerData = CustomerDataPolicy(
    email = CollectCustomerField(enabled = true, required = true),
)
```

## Status monitoring

| Parameter | Type | Default | Description |
|---|---|---|---|
| `statusJsonMapping` | `PaymentStatusJsonMapping` | current provider codes | How to interpret the page's status response. |
| `pagePollIntervalMillis` | `Long` | `1_000` | The interval between payment-page reads. |
| `captureWarningDelayMillis` | `Long` | `15_000` | After this long with no response captured at all, the SDK writes a warning to the log. |

### `PaymentStatusJsonMapping`

| Field | Default |
|---|---|
| `statusFieldNames` | `status`, `transaction_status`, `payment_status`, `state` |
| `transactionIdFieldNames` | `transaction`, `transaction_id`, `transactionId`, `id` |
| `choosingMethodValues` | `active` |
| `pendingValues` | `PNDNG`, `pending` |
| `successValues` | `SETLD`, `settled`, `success`, `paid` |
| `failedValues` | `FAILD`, `EXPRD`, `RJCTD`, `DECLN`, `failed`, `failure`, `error`, `rejected`, `expired` |
| `cancelledValues` | `CNCLD`, `cancelled`, `canceled` |
| `statusUrlFragments` | `payment-link`, `transaction` |
| `methodFieldNames`, `vaNumberFieldNames`, `channelValues` | Used for simulated settlement only |

Every default value is available as a `DEFAULT_*` constant on the companion
object, so you can extend it without retyping the rest:

```kotlin
PaymentStatusJsonMapping(
    successValues = PaymentStatusJsonMapping.DEFAULT_SUCCESS_VALUES + "CMPLT",
)
```

Value matching is case-insensitive.

## Derived properties

| Property | Description |
|---|---|
| `isSimulated` | `true` when `baseUrl` is empty (prototype mode). |
| `hasLoginCredentials` | `true` when both `email` and `password` are set. |

## Production example

```kotlin
OnlinePaymentConfig(
    baseUrl = "https://pay-api.tokoanda.co.id",
    apiKey = deviceKey,
    email = account.email,
    password = account.password,
    loopbackUrlReplacement = "",
    allowedPaymentMethods = setOf(OnlinePaymentChannel.VIRTUAL_ACCOUNT, OnlinePaymentChannel.QRIS),
    appearance = PaymentLinkAppearance(logoUrl = "https://tokoanda.co.id/logo.png"),
)
```

From Swift, use [`onlinePaymentConfigOf`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/swift-bridge#onlinepaymentconfigof).
