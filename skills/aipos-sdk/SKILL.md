---
name: aipos-sdk
description: Integrate the AI POS SDK (com.weekendinc.aipos, Kotlin Multiplatform) into Android, iOS/Swift, or KMP apps — cart, card payment on a MineSec terminal, online payment (VA/QRIS) in a WebView, AI product suggestions, and the cashier agent. Use when code imports com.weekendinc.aipos or AIPosSDK, or when the user asks to add POS, payment, product advisor, or cashier-agent features with this SDK.
---

# AI POS SDK (version 0.1.1)

Kotlin Multiplatform SDK for point-of-sale apps. Full docs for agents:

- Index: https://dev-mobile-wknd.github.io/aipos-sdk-docs/llms.txt
- Everything in one file: https://dev-mobile-wknd.github.io/aipos-sdk-docs/llms-full.txt
- Any docs page as Markdown: append `.md` to its URL.

Read the relevant page before writing code for a feature not covered here. Do not guess
API names — if a method isn't listed here or in the docs, it doesn't exist.

## Pick the artifact

Group `com.weekendinc.aipos`, every artifact at the **same** version. Declare only one;
the rest come transitively.

| Artifact | Entry point | Use for |
|---|---|---|
| `aipos-sdk` | `AIPosSDK` | Everything; one cart shared by advisor, agent, payment. **Default choice.** |
| `aipos-advisor` | `AdvisorClient` | AI product suggestions only (no cart) |
| `aipos-payment` | `PaymentClient` | Cart + card payment on the terminal, no LLM key needed |
| `aipos-payment-online` | `OnlinePaymentClient` | Cart + online payment (VA/QRIS) in a WebView, no LLM key needed |

`aipos-payment-online` alone does not declare `INTERNET` — add it to the app manifest.

## Install

**Android** — the SDK is on private GitHub Packages. Credentials go in
`~/.gradle/gradle.properties`, never in the project:

```properties
GITHUB_PACKAGES_LOGIN=<github username>
GITHUB_PACKAGES_TOKEN=<PAT with read:packages>
```

```kotlin
// settings.gradle.kts → dependencyResolutionManagement.repositories
maven {
    name = "AIPosSDK"
    url = uri("https://maven.pkg.github.com/dev-mobile-wknd/aipos-sdk-android")
    credentials {
        username = providers.gradleProperty("GITHUB_PACKAGES_LOGIN").get()
        password = providers.gradleProperty("GITHUB_PACKAGES_TOKEN").get()
    }
}

// app/build.gradle.kts
implementation("com.weekendinc.aipos:aipos-sdk:0.1.1")
```

**iOS (pure Swift)** — Swift Package Manager, URL
`https://github.com/dev-mobile-wknd/swift-aipos-sdk`, then `import AIPosSDK`. The app
target **must** set `OTHER_LDFLAGS = $(inherited) -lc++`, otherwise linking fails with
missing `std::` symbols. Card payment is not available on iOS.

**KMP shared module** — `api("com.weekendinc.aipos:aipos-sdk:0.1.1")` in
`commonMain` and `export(...)` every aipos artifact in the iOS framework block. Needs
Kotlin 2.3.21+.

## Assemble the SDK (Android)

```kotlin
// Application.onCreate — initialize the card terminal once
appScope.launch { AiPosAndroid.initPlatform(this@PosApplication) }

// Activity.onCreate — REQUIRED before any card payment
AiPosAndroid.attachActivity(this)

// One instance per process (e.g. in a ViewModel)
val sdk: AIPosSDK = AIPosSDK.Builder()
    .llmApiKey(BuildConfig.LLM_API_KEY)          // or .llmProxyBaseUrl(...) — one is required
    .merchantInfo(MerchantInfo(id = "m1", name = "Store", address = "...", terminalId = "TID001", mid = "MID123"))
    .productCatalog(MutableProductCatalog(products))
    .onlinePayment(OnlinePaymentConfig(/* named args only */))  // optional; default = beta backend
    .android(application)                        // or .ios() on iOS
    .build()                                     // throws IllegalArgumentException if config is missing

// When done: sdk.close() is suspend — launch it in a scope that outlives the ViewModel.
```

## Rules that prevent the common bugs

- **Money**: build prices with `Money.fromRupiah(13_999_000)`. The `Money(...)`
  constructor takes **cents**. In Swift, money arrives as `Int64` cents — format with
  `IosDomainFactoryKt.formatMoneyCents(cents:)`.
- **Errors**: fallible calls return `PosResult<T>` (`PosResult.Success` /
  `PosResult.Failure` with a display-safe `message`); they don't throw. Handle both
  branches.
- **State**: observe `Flow`s — `observeCatalog()`, `observeCart()`,
  `observePaymentState()`, `observeSuggestions()`, `onlinePaymentStatus` — and render
  from them instead of keeping your own copies.
- **Cart**: `addToCart(productId, quantity)`, `updateCartQuantity` (≤ 0 removes),
  `removeFromCart`, `clearCart`. Stock is validated. A successful payment clears the cart.
- **Card payment**: `processPayment()` returns a `Flow` — `collect()` it.
  `PaymentState`: `Idle`, `WaitingTap`, `Processing`, `Success`, `Failed`.
- **Online payment**: `startOnlinePayment(note)` → `WebPaymentSession`; load
  `session.paymentUrl` in `AiposPaymentWebView` with `AiposPaymentWebViewClient(sdk)`
  and **always** call `sdk.attachPaymentPageProbe(webView.paymentPageProbe())`, or the
  status stays stuck at `ChoosingMethod`. Create the WebView once (`remember`), not on
  every recomposition.
- **`OnlinePaymentConfig`**: always named arguments (15 params). `baseUrl` and
  `llmProxyBaseUrl` go **without** `/v1`. `baseUrl = ""` is prototype mode (no HTTP).
  Set `loopbackUrlReplacement = ""` in production.
- **AI**: `pushTranscript(text)` feeds speech into suggestions (debounced ~1.5 s);
  `suspend sendMessage(text)` drives the cashier agent, which can modify the cart.
- **Secrets**: `llmApiKey` bundled in an APK is extractable — dev/demo only. Production
  uses `llmProxyBaseUrl` pointing at your own server.
- **Internal API**: never use declarations marked `@InternalAiPosApi`.

## Swift specifics

- Kotlin default arguments are required from Swift (e.g. pass `profileId:
  MerchantInfo.companion.DEFAULT_PROFILE_ID`).
- Build products with `IosDomainFactoryKt.productOf(...)` (price in whole Rupiah) and
  online config with `IosOnlinePaymentConfigFactoryKt.onlinePaymentConfigOf(...)`.
- `suspend` → `try await`. Results are `PosResultSuccess` / `PosResultFailure` — downcast with `as?`.
- Observe flows with `FlowSubscriptionKt.subscribe(flow) { value in ... }` and cancel
  the returned `FlowSubscription`; values are `Any?`, downcast them.

## `build()` error messages (Indonesian)

| Message | Fix |
|---|---|
| `MerchantInfo wajib diisi` | call `.merchantInfo(...)` |
| `Sumber katalog wajib diisi` | call `.productCatalog(...)` |
| `Platform belum diatur` | call `.android(context)` / `.ios()` |
| `Salah satu dari llmApiKey(...) atau llmProxyBaseUrl(...) wajib diisi` | set one of them (`"dummy"` works for non-AI testing) |

## Where to read more

- Configuration: https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/configuration.md
- Online payment: https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/online-payment.md
- iOS: https://dev-mobile-wknd.github.io/aipos-sdk-docs/platform/ios.md
- API reference: https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference.md
- Troubleshooting: https://dev-mobile-wknd.github.io/aipos-sdk-docs/help/troubleshooting.md
