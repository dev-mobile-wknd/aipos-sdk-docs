# AdvisorClient

```kotlin
class AdvisorClient
```

Product suggestions without any other feature. Artifact: `com.weekendinc.aipos:aipos-advisor`.
Package: `com.weekendinc.aipos.advisor`.

Doesn't bring in MineSec, the NFC permission, or the cashier agent. Has no cart.

## Builder

```kotlin
val advisor = AdvisorClient.Builder()
    .llmApiKey(key)            // or .llmProxyBaseUrl(url)
    .productCatalog(catalog)
    .build()
```

| Method | Required | Description |
|---|---|---|
| `llmApiKey(key: String)` | one of | Direct OpenAI key. |
| `llmProxyBaseUrl(url: String)` | one of | An OpenAI-compatible endpoint, without `/v1`. |
| `productCatalog(source: ProductCatalogSource)` | yes | Only products here may be suggested. |
| `build(): AdvisorClient` | — | Doesn't need a `Context` or a platform. |

## Members

| Member | Type | Description |
|---|---|---|
| `pushTranscript(text: String)` | `Unit` | Submit one final sentence. |
| `setInterimTranscript(text: String)` | `Unit` | A partial sentence still being recognized. |
| `requestSuggestions()` | `Unit` | Analyze now. |
| `observeSuggestions()` | `Flow<List<ProductSuggestion>>` | Latest suggestions. |
| `observeBuyerIntent()` | `Flow<String>` | A summary of the customer's needs. |
| `observeAdvisorState()` | `Flow<AdvisorState>` | `Idle`, `Thinking`, `Suggesting`. |
| `observeTranscript()` | `Flow<List<String>>` | Every sentence submitted so far. |
| `observeInterimTranscript()` | `Flow<String>` | The last partial sentence. |
| `observeAdvisorError()` | `Flow<String?>` | The latest failure. |
| `observeCatalog()` | `Flow<List<Product>>` | The catalog contents. |
| `clear()` | `Unit` | Clear the transcript and suggestions. |
| `close()` | `Unit` | Release resources. Not a `suspend` function. |

See the full guide at [Product suggestions](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/advisor).
