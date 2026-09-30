# Configuration

All configuration is provided through the builder when the SDK is assembled. `build()`
validates required configuration and throws `IllegalArgumentException` if something is
missing — wiring mistakes surface when the app opens, not when a customer is already
waiting at the register.

## Builder summary

| Method | `AIPosSDK` | `AdvisorClient` | `PaymentClient` | `OnlinePaymentClient` |
|---|:---:|:---:|:---:|:---:|
| `.merchantInfo(...)` | required | — | required | required |
| `.productCatalog(...)` | required | required | required | required |
| `.llmApiKey(...)` / `.llmProxyBaseUrl(...)` | one required | one required | — | — |
| `.onlinePayment(...)` / `.config(...)` | optional | — | — | optional |
| `.android(context)` / `.ios()` | required | — | required | required |

## Merchant identity

`MerchantInfo` is printed on the receipt and sent to the payment gateway.

```kotlin
val merchant = MerchantInfo(
    id = "merchant_001",                     // your internal id
    name = "Jaya Gadget Store",              // printed on the receipt
    address = "Jl. Sudirman No. 1, Jakarta",
    terminalId = "TID001",                   // Terminal ID from the acquirer
    mid = "MID123456",                       // Merchant ID from the acquirer
    profileId = "prof_xxxxxxxx",             // your MineSec profile
)
```

| Field | Description |
|---|---|
| `id` | Merchant identity in your own system. |
| `name` | Store name printed on the receipt. |
| `address` | Store address. |
| `terminalId` | Terminal ID (TID) from the acquirer. |
| `mid` | Merchant ID (MID) from the acquirer. |
| `profileId` | MineSec terminal profile from the MineSec dashboard. Defaults to `MerchantInfo.DEFAULT_PROFILE_ID`, a **test environment** profile. Production merchants must supply their own profile. |

## Language model

AI features (product suggestions and the cashier agent) need access to a language
model. Choose one of two ways:

**Development: direct API key**

```kotlin
AIPosSDK.Builder()
    .llmApiKey(BuildConfig.LLM_API_KEY)
    // ...
```

**Production: via a proxy**

```kotlin
AIPosSDK.Builder()
    .llmProxyBaseUrl("https://api.tokoanda.com/llm")
    // ...
```

| Situation | What the SDK does |
|---|---|
| Only `llmApiKey` is set | Calls OpenAI directly with that key. |
| Only `llmProxyBaseUrl` is set | Calls the proxy without a key. |
| Both are set | Calls the **proxy**, and `llmApiKey` is sent as the authentication token to the proxy. |
| Both are empty | `build()` fails: *"Salah satu dari llmApiKey(...) atau llmProxyBaseUrl(...) wajib diisi"* (one of llmApiKey(...) or llmProxyBaseUrl(...) is required). |

:::danger[An API key bundled in the app can be extracted]
Anyone who downloads your APK can extract a key bundled inside it. Use `llmApiKey`
only for development and demos. For production, see [LLM proxy](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guides/llm-proxy).
:::

:::warning[No `/v1`]
Set `llmProxyBaseUrl` **without** a trailing `/v1`. The SDK appends
`v1/chat/completions` itself, so `https://api.tokoanda.com/llm/v1` would produce a
duplicated URL and end in a 404.
:::

## Product catalog

Required for every entry point. See [Product catalog](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/catalog).

```kotlin
.productCatalog(MutableProductCatalog(storeProducts))
```

## Online payment

Optional. Without this call, the SDK uses the default `OnlinePaymentConfig()`, which
points at the **beta environment** — fine for trying the flow, not for real transactions.

```kotlin
AIPosSDK.Builder()
    .onlinePayment(
        OnlinePaymentConfig(
            baseUrl = "https://pay-api.tokoanda.co.id",   // without /v1
            apiKey = BuildConfig.POS_BACKEND_API_KEY,
            email = "cashier@tokoanda.co.id",
            password = passwordFromSecureStorage,
            loopbackUrlReplacement = "",                  // disable in production
        )
    )
```

:::warning[Always use named arguments]
`OnlinePaymentConfig` has fifteen parameters. Calling it positionally is easy to get
wrong without a compile error — for instance, a password landing in the email slot.
:::

Three modes you can choose from:

| Mode | How to enable it | Behavior |
|---|---|---|
| **Prototype** | `OnlinePaymentConfig(baseUrl = "")` | No HTTP requests at all. The SDK uses a sample payment page. For building the UI. |
| **Beta** | Don't call `.onlinePayment(...)`, or only set `apiKey` | The SDK's shared beta backend. For integration testing. |
| **Production** | Set your own `baseUrl`, `apiKey`, `email`, `password` | Your own merchant backend. |

The full parameter list is in [OnlinePaymentConfig](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/online-payment-config).

## Storing secrets

Don't write keys directly in code. A simple pattern for Android:

```properties
# local.properties — gitignored by default
LLM_API_KEY=sk-proj-...
POS_BACKEND_API_KEY=...
```

```kotlin title="app/build.gradle.kts"
import java.util.Properties

val localProps = Properties().apply {
    rootProject.file("local.properties").takeIf { it.exists() }?.inputStream()?.use(::load)
}

android {
    buildFeatures { buildConfig = true }
    defaultConfig {
        buildConfigField("String", "LLM_API_KEY", "\"${localProps.getProperty("LLM_API_KEY", "")}\"")
        buildConfigField("String", "POS_BACKEND_API_KEY", "\"${localProps.getProperty("POS_BACKEND_API_KEY", "")}\"")
    }
}
```

This keeps keys out of Git, **but the key still ends up bundled in the APK**. For
production, move the key to a server — see [Production checklist](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guides/production-checklist).

## Error messages from `build()`

The SDK's error messages are in Indonesian, matching its primary market:

| Message | Cause |
|---|---|
| `MerchantInfo wajib diisi — panggil merchantInfo(...)` | `.merchantInfo(...)` hasn't been called. |
| `Sumber katalog wajib diisi — panggil productCatalog(...)` | `.productCatalog(...)` hasn't been called. |
| `Platform belum diatur — panggil .android(context) di Android atau .ios() di iOS` | Forgot to call `.android(context)` or `.ios()`. |
| `Salah satu dari llmApiKey(...) atau llmProxyBaseUrl(...) wajib diisi` | Both language-model paths are empty. |
