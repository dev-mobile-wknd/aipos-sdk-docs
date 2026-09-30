# AIPosSDK

```kotlin
class AIPosSDK : WebPaymentUrlListener
```

Titik masuk tunggal untuk semua fitur. Artifact: `com.weekendinc.aipos:aipos-sdk`.

## Builder

```kotlin
val sdk = AIPosSDK.Builder()
    .llmApiKey(key)                 // or .llmProxyBaseUrl(url)
    .merchantInfo(merchant)
    .productCatalog(catalog)
    .onlinePayment(config)          // optional
    .android(context)               // or .ios()
    .build()
```

| Metode | Wajib | Keterangan |
|---|---|---|
| `llmApiKey(key: String)` | salah satu | Kunci OpenAI langsung. Untuk pengembangan. Bila proxy juga diisi, dikirim sebagai token ke proxy. |
| `llmProxyBaseUrl(url: String)` | salah satu | Endpoint kompatibel OpenAI, **tanpa** `/v1`. Didahulukan bila keduanya diisi. |
| `merchantInfo(info: MerchantInfo)` | ya | Identitas merchant. |
| `productCatalog(source: ProductCatalogSource)` | ya | Sumber katalog produk. |
| `onlinePayment(config: OnlinePaymentConfig)` | tidak | Bawaan: `OnlinePaymentConfig()` (lingkungan beta). |
| `android(context: Context)` | ya (Android) | Extension di `com.weekendinc.aipos`. |
| `ios()` | ya (iOS) | Extension di `com.weekendinc.aipos`. |
| `build(): AIPosSDK` | — | Melempar `IllegalArgumentException` bila konfigurasi wajib belum lengkap. |

## Katalog

| Anggota | Tipe | Keterangan |
|---|---|---|
| `observeCatalog()` | `Flow<List<Product>>` | Isi katalog dari `productCatalog(...)`. |

## Penyaran produk

| Anggota | Tipe | Keterangan |
|---|---|---|
| `pushTranscript(text: String)` | `Unit` | Setor satu kalimat final. Aman dipanggil sesering apa pun. |
| `setInterimTranscript(text: String)` | `Unit` | Potongan kalimat yang masih dikenali. |
| `requestSuggestions()` | `Unit` | Analisis sekarang, tanpa menunggu percakapan mereda. |
| `clearAdvisor()` | `Unit` | Kosongkan transkrip dan saran. Keranjang tidak disentuh. |
| `observeSuggestions()` | `Flow<List<ProductSuggestion>>` | Saran terbaru, urut dari yang paling relevan. |
| `observeBuyerIntent()` | `Flow<String>` | Ringkasan kebutuhan pelanggan. |
| `observeAdvisorState()` | `Flow<AdvisorState>` | `Idle`, `Thinking`, `Suggesting`. |
| `observeTranscript()` | `Flow<List<String>>` | Semua kalimat yang sudah disetor. |
| `observeInterimTranscript()` | `Flow<String>` | Potongan kalimat terakhir. |
| `observeAdvisorError()` | `Flow<String?>` | Kegagalan terakhir; `null` bila tidak ada. |

## Agent kasir

| Anggota | Tipe | Keterangan |
|---|---|---|
| `suspend sendMessage(message: String)` | `String` | Kirim perintah; kembali setelah agent selesai. |
| `observeMessages()` | `Flow<List<AgentMessage>>` | Riwayat percakapan. |
| `observeProcessing()` | `Flow<Boolean>` | `true` selama agent memproses pesan. |

## Keranjang

| Anggota | Tipe | Keterangan |
|---|---|---|
| `observeCart()` | `Flow<Cart>` | Isi keranjang. |
| `suspend addToCart(productId: ProductId, quantity: Int = 1)` | `PosResult<Cart>` | Tambah dengan validasi stok. |
| `suspend updateCartQuantity(productId: ProductId, quantity: Int)` | `PosResult<Cart>` | Jumlah ≤ 0 menghapus baris. |
| `suspend removeFromCart(productId: ProductId)` | `PosResult<Cart>` | Hapus baris. |
| `suspend clearCart()` | `Unit` | Kosongkan keranjang. |

## Pembayaran kartu

| Anggota | Tipe | Keterangan |
|---|---|---|
| `processPayment()` | `Flow<PaymentState>` | Tagih total keranjang. Stream berakhir di `Success` atau `Failed`. |
| `suspend cancelPayment()` | `Unit` | Batalkan pembayaran yang menunggu tap. |
| `observePaymentState()` | `Flow<PaymentState>` | Status terkini, termasuk pembayaran yang dijalankan agent. |
| `observeTransactions(limit: Int = 20)` | `Flow<List<Transaction>>` | Riwayat transaksi di perangkat. |

## Pembayaran online

| Anggota | Tipe | Keterangan |
|---|---|---|
| `onlinePaymentStatus` | `StateFlow<WebPaymentStatus>` | Status sesi yang berjalan; `Idle` bila tidak ada. |
| `suspend startOnlinePayment(note: String? = null)` | `PosResult<WebPaymentSession>` | Buka sesi untuk isi keranjang. |
| `attachPaymentPageProbe(probe: PaymentPageProbe)` | `Unit` | **Wajib.** Sambungkan WebView yang memuat halaman pembayaran. |
| `onWebViewUrlChanged(url: String)` | `Unit` | Laporkan URL WebView. Sudah dilakukan `AiposPaymentWebViewClient`. |
| `onWebViewLoadFailed(reason: String)` | `Unit` | Laporkan halaman gagal dimuat. |
| `addOnlinePaymentListener(onStatus: PaymentStatusListener)` | `PaymentListenerRegistration` | Callback alternatif `Flow`. Batalkan dengan `cancel()`. |
| `cancelOnlinePayment()` | `Unit` | Batalkan sesi. Keranjang tidak dikosongkan. |

## Siklus hidup

| Anggota | Tipe | Keterangan |
|---|---|---|
| `suspend resetSession()` | `Unit` | Kosongkan percakapan, saran, keranjang, status pembayaran, dan sesi online. |
| `suspend close()` | `Unit` | Lepaskan semua sumber daya. Instance tidak boleh dipakai lagi. |
