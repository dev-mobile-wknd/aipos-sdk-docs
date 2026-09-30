# OnlinePaymentClient

```kotlin
class OnlinePaymentClient : WebPaymentUrlListener
```

Keranjang dan pembayaran online tanpa terminal dan tanpa fitur AI. Artifact:
`com.weekendinc.aipos:aipos-payment-online`. Paket: `com.weekendinc.aipos.payment.online`.

## Builder

```kotlin
val online = OnlinePaymentClient.Builder()
    .merchantInfo(merchant)
    .productCatalog(catalog)
    .config(OnlinePaymentConfig(apiKey = key))   // optional
    .android(context)                             // or .ios()
    .build()
```

| Metode | Wajib | Keterangan |
|---|---|---|
| `merchantInfo(info: MerchantInfo)` | ya | Identitas merchant. |
| `productCatalog(source: ProductCatalogSource)` | ya | Harga dan stok yang ditagihkan. |
| `config(config: OnlinePaymentConfig)` | tidak | Bawaan: `OnlinePaymentConfig()` (lingkungan beta). |
| `android(context: Context)` | ya (Android) | Extension di `com.weekendinc.aipos.payment.online`. |
| `ios()` | ya (iOS) | Extension di `com.weekendinc.aipos.payment.online`. |
| `build(): OnlinePaymentClient` | — | Melempar `IllegalArgumentException` bila konfigurasi wajib belum lengkap. |

## Anggota

| Anggota | Padanan di `AIPosSDK` | Tipe |
|---|---|---|
| `observeCatalog()` | sama | `Flow<List<Product>>` |
| `observeCart()` | sama | `Flow<Cart>` |
| `suspend addToCart(productId: ProductId, quantity: Int = 1)` | sama | `PosResult<Cart>` — validasi stok |
| `suspend updateCartQuantity(productId: ProductId, quantity: Int)` | sama | `PosResult<Cart>` — jumlah ≤ 0 menghapus baris |
| `suspend removeFromCart(productId: ProductId)` | sama | `PosResult<Cart>` |
| `suspend clearCart()` | sama | `Unit` |
| `status` | `onlinePaymentStatus` | `StateFlow<WebPaymentStatus>` |
| `suspend startSession(note: String? = null)` | `startOnlinePayment(note)` | `PosResult<WebPaymentSession>` |
| `attachPaymentPageProbe(probe: PaymentPageProbe)` | sama | `Unit` |
| `onWebViewUrlChanged(url: String)` | sama | `Unit` |
| `onWebViewLoadFailed(reason: String)` | sama | `Unit` |
| `addStatusListener(onStatus: PaymentStatusListener)` | `addOnlinePaymentListener(onStatus)` | `PaymentListenerRegistration` |
| `cancelSession()` | `cancelOnlinePayment()` | `Unit` |
| `resetSession()` | — | `Unit` — kembali ke `Idle`; keranjang tidak disentuh |
| `close()` | `close()` | `Unit` |

## Adapter WebView

| Deklarasi | Platform | Keterangan |
|---|---|---|
| `AiposPaymentWebView(context)` | Android | `WebView` yang sudah disiapkan dan tetap bisa digulir di bottom sheet |
| `AiposPaymentWebViewClient(listener: WebPaymentUrlListener)` | Android | Meneruskan perubahan URL dan kegagalan muat ke SDK |
| `WebView.prepareForAiposPayment()` | Android | Menyalakan JavaScript, DOM storage, dan penyesuaian viewport pada `WebView` biasa |
| `WebView.paymentPageProbe(): PaymentPageProbe` | Android | Probe untuk `attachPaymentPageProbe` |
| `WKWebView.paymentPageProbe(): PaymentPageProbe` | iOS | Di Swift: `IosPaymentPageProbeKt.paymentPageProbe(webView)` |
| `AiposPaymentNavigationDelegate(listener)` | iOS (Kotlin saja) | Tidak bisa dipakai dari Swift — tulis `WKNavigationDelegate` sendiri |

Lihat panduan lengkap di [Pembayaran online](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/online-payment).
