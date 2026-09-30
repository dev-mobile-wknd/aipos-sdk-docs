# Troubleshooting

Note: the SDK's own runtime error and log messages are in Indonesian, matching
its primary market. The tables below keep those literal messages as-is so you can
match them against what you actually see, with the cause and fix in English.

## Installation and build

### `Could not find com.theminesec.sdk:headless...`

The MineSec repository isn't registered, or its credentials are wrong.

1. Make sure `MINESEC_REGISTRY_LOGIN` and `MINESEC_REGISTRY_TOKEN` are in
   `~/.gradle/gradle.properties`.
2. Make sure the block `maven { url = uri("https://maven.pkg.github.com/theminesec/ms-registry-client") }`
   is in `settings.gradle.kts` — see [Installation](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/installation#minesec-repository-card-payment).
3. An expired GitHub Packages token also produces this message. Ask MineSec for a
   new token.

### `Dependency ... requires compileSdk 36`

Raise the app module's `compileSdk` to 36.

### `Manifest merger failed: uses-sdk:minSdkVersion 24 cannot be smaller than version 26`

The SDK requires `minSdk = 26`.

### Crash `JNI DETECTED ERROR ... SimpleLoggerFactory`

There are two SLF4J backends on the classpath: `slf4j-simple` and
`logback-android` (brought in by MineSec). MineSec's native code requires
logback. Find what's pulling in `slf4j-simple`:

```bash
./gradlew :app:dependencies --configuration releaseRuntimeClasspath | grep slf4j-simple
```

Then exclude it from your app:

```kotlin
configurations.configureEach {
    exclude(group = "org.slf4j", module = "slf4j-simple")
}
```

### Log `No SLF4J providers were found`

Not an error. An app that doesn't use MineSec has no SLF4J logging backend, so
internal logs aren't shown. Add your own backend if needed, e.g.
`com.github.tony19:logback-android`.

### iOS linking fails with `std::` symbols not found

Add `-lc++` to `OTHER_LDFLAGS`. See [iOS integration](https://dev-mobile-wknd.github.io/aipos-sdk-docs/platform/ios#b-swift-package-manager-for-swift-apps).

## Initialization

### `IllegalArgumentException` on `build()`

Read the message — each one names the builder method that hasn't been called.
The full list is in [Results & errors](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/results-and-errors#builder).

### The app hangs when adding to the cart

Your `ProductCatalogSource` never emits a value. A flow must emit its first value
promptly — see [Product catalog](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/catalog#option-2-your-own-implementation).

## Card payment

| Symptom | Cause | Fix |
|---|---|---|
| `PaymentActivityBridge belum di-attach` (not attached) | `attachActivity` wasn't called, or was called in `onResume` | Call `AiPosAndroid.attachActivity(this)` in `onCreate`, before `setContent` |
| Payment always succeeds within ~3 seconds, `[SIMULASI]` message | The installed artifact is the simulator variant | Use an artifact built with MineSec |
| `Profile ID MineSec belum diatur` (MineSec profile ID not set) | `MerchantInfo.profileId` is empty | Fill it in from the MineSec dashboard |
| `HeadlessActivity must be declared with android:launchMode=singleTask` | Your app's manifest overrides the SDK's Activity declaration | Remove `tools:replace` for `AIPosHeadlessActivity` |
| `initPlatform` fails about the license | The license file isn't in `src/main/assets/`, or its name differs | Check the name passed to `licenseName` |
| `Pembayaran contactless belum didukung di iOS` (contactless payment not supported on iOS) | Called on iOS | Use online payment on iOS |

## Online payment

Filter the logs first — many issues are visible right away:

```bash
adb logcat | grep AiPos/online-payment
```

| Symptom | Cause | Fix |
|---|---|---|
| Status stuck at `ChoosingMethod` | `attachPaymentPageProbe` hasn't been called | Call `sdk.attachPaymentPageProbe(webView.paymentPageProbe())` after the WebView is created |
| Log: *"no page response has been captured"* | The probe is attached to the wrong WebView, or the page uses a Service Worker / cross-origin iframe / WebSocket | Make sure the probe comes from the same WebView loading `paymentUrl` |
| Payload captured but status is `Unknown(rawStatus = ...)` | The provider uses a new status code | Add the code to `statusJsonMapping` — see [Online payment](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/online-payment#when-a-provider-changes-its-status-codes) |
| A status endpoint is visible in the network tab but ignored | Its address doesn't contain a recognized fragment | Add the fragment to `statusJsonMapping.statusUrlFragments` |
| No request to `/v1/payment-links` | `baseUrl` is empty → prototype mode | Set `baseUrl`; the log when the SDK is assembled names the active mode |
| *"payment login credentials not set"* | `email` or `password` is empty | Fill in both |
| *"payment login rejected by server"* | Wrong email/password, or a wrong `baseUrl` making `/v1/login` return 404 | Check the email printed in the mode log, then the credentials |
| *"payment access denied even after re-login"* | The token is already fresh; the issue is `apiKey` or account permissions | Check `apiKey` and the account's role on the backend |
| Blank/white page | WebView JavaScript is off | Use `AiposPaymentWebView`, or call `prepareForAiposPayment()` |
| Page fails to load, the address contains `localhost` | The backend returns a loopback address | Set `loopbackUrlReplacement` (beta), or fix `public_url` on the backend (production) |
| Page reloads every time the status changes | The WebView is recreated on every recomposition | Wrap WebView creation in `remember` |
| Page can't be scrolled inside a bottom sheet | The sheet's gesture is swallowing the touch | Use `AiposPaymentWebView` |
| `startOnlinePayment` fails with *"Keranjang masih kosong"* (cart is still empty) despite using `OnlinePaymentClient` | The cart wasn't filled before opening the session | Call `addToCart(...)` first — available on `OnlinePaymentClient` since 0.1.1 |
| On iOS, page navigation isn't detected | `WKWebView` doesn't call the delegate for `history.pushState` | Add KVO on `webView.url` |

## AI features

| Symptom | Cause | Fix |
|---|---|---|
| Suggestions never appear | `pushTranscript` was never called, only `setInterimTranscript` | Submit the final sentence via `pushTranscript` |
| `observeAdvisorError()` has a 401 message | The OpenAI key or proxy token is wrong | Check `llmApiKey` / the proxy |
| 404 from the proxy | `llmProxyBaseUrl` ends with `/v1` | Remove `/v1` |
| Suggestions are always empty despite a matching product existing | The model named a product that isn't an exact catalog match, so it was dropped | Fill in the product's `name` and `description` more completely |
| Suggestions target the previous customer | The old transcript wasn't cleared | Call `clearAdvisor()` or `resetSession()` when the customer changes |
| On iOS, suggestions are never analyzed | The recognizer never produces a final result | Determine sentence end with a silence delay — see [Speech-to-text](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guides/speech-to-text#determining-the-end-of-a-sentence) |
