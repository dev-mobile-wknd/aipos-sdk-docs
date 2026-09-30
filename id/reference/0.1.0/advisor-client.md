# AdvisorClient

```kotlin
class AdvisorClient
```

Penyaran produk tanpa fitur lain. Artifact: `com.weekendinc.aipos:aipos-advisor`.
Paket: `com.weekendinc.aipos.advisor`.

Tidak membawa MineSec, izin NFC, maupun agent kasir. Tidak punya keranjang.

## Builder

```kotlin
val advisor = AdvisorClient.Builder()
    .llmApiKey(key)            // or .llmProxyBaseUrl(url)
    .productCatalog(catalog)
    .build()
```

| Metode | Wajib | Keterangan |
|---|---|---|
| `llmApiKey(key: String)` | salah satu | Kunci OpenAI langsung. |
| `llmProxyBaseUrl(url: String)` | salah satu | Endpoint kompatibel OpenAI, tanpa `/v1`. |
| `productCatalog(source: ProductCatalogSource)` | ya | Hanya produk di sini yang boleh disarankan. |
| `build(): AdvisorClient` | — | Tidak membutuhkan `Context` atau platform. |

## Anggota

| Anggota | Tipe | Keterangan |
|---|---|---|
| `pushTranscript(text: String)` | `Unit` | Setor satu kalimat final. |
| `setInterimTranscript(text: String)` | `Unit` | Potongan kalimat yang masih dikenali. |
| `requestSuggestions()` | `Unit` | Analisis sekarang. |
| `observeSuggestions()` | `Flow<List<ProductSuggestion>>` | Saran terbaru. |
| `observeBuyerIntent()` | `Flow<String>` | Ringkasan kebutuhan pelanggan. |
| `observeAdvisorState()` | `Flow<AdvisorState>` | `Idle`, `Thinking`, `Suggesting`. |
| `observeTranscript()` | `Flow<List<String>>` | Semua kalimat yang sudah disetor. |
| `observeInterimTranscript()` | `Flow<String>` | Potongan kalimat terakhir. |
| `observeAdvisorError()` | `Flow<String?>` | Kegagalan terakhir. |
| `observeCatalog()` | `Flow<List<Product>>` | Isi katalog. |
| `clear()` | `Unit` | Kosongkan transkrip dan saran. |
| `close()` | `Unit` | Lepaskan sumber daya. Bukan fungsi `suspend`. |

Lihat panduan lengkap di [Penyaran produk](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/advisor).
