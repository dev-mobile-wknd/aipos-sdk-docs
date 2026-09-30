# Changelog

All `com.weekendinc.aipos:*` artifacts are released together at one version number.

## 0.1.0

First release.

### Features

- **Product suggestions** — `AdvisorClient` and `AIPosSDK.pushTranscript` / `observeSuggestions`.
- **Cashier agent** — `AIPosSDK.sendMessage` with actions for product search, cart,
  card payment, receipts, and transaction history.
- **Cart** with stock validation and local storage.
- **Contactless card payment** on Android via MineSec Headless SDK 1.3, with a
  built-in simulator.
- **Online payment** — Virtual Account, QRIS, and card through a payment page in a
  WebView, with automatic status reading (`attachPaymentPageProbe`), automatic
  token login, and configurable status-code mapping.
- **Kotlin Multiplatform** — Android and iOS (`iosArm64`, `iosSimulatorArm64`,
  `iosX64`), with Swift bridge functions.

### Known limitations

- Contactless card payment isn't supported on iOS yet.
- `OnlinePaymentClient` doesn't provide cart functionality yet; use `AIPosSDK` for
  online payment.
- The online payment paid status isn't verified with a server by the SDK.
- The SDK calls `POST /v1/transactions/simulate-paid` about three seconds after
  the `Pending` status when `baseUrl` is set.
- Provider status codes for failure, expiry, and cancellation haven't been
  verified in the field; unrecognized codes show up as `WebPaymentStatus.Unknown`.
