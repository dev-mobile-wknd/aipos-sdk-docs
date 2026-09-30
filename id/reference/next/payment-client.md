# PaymentClient

```kotlin
class PaymentClient
```

Keranjang dan pembayaran kartu tanpa fitur AI. Artifact: `com.weekendinc.aipos:aipos-payment`.
Paket: `com.weekendinc.aipos.payment`.

Tidak membawa klien OpenAI maupun izin internet.

## Builder

```kotlin
val payment = PaymentClient.Builder()
    .merchantInfo(merchant)
    .productCatalog(catalog)
    .android(context)          // or .ios()
    .build()
```

| Metode | Wajib | Keterangan |
|---|---|---|
| `merchantInfo(info: MerchantInfo)` | ya | Identitas merchant. |
| `productCatalog(source: ProductCatalogSource)` | ya | Stok keranjang divalidasi terhadap katalog ini. |
| `android(context: Context)` | ya (Android) | Extension di `com.weekendinc.aipos.payment`. |
| `ios()` | ya (iOS) | Extension di `com.weekendinc.aipos.payment`. Pembayaran kartu tidak didukung di iOS. |
| `build(): PaymentClient` | — | Melempar `IllegalArgumentException` bila konfigurasi wajib belum lengkap. |

Di Android, [`AiPosAndroid.initPlatform`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/reference/aipos-android) dan
`AiPosAndroid.attachActivity` tetap wajib dipanggil.

## Anggota

| Anggota | Tipe | Keterangan |
|---|---|---|
| `observeCatalog()` | `Flow<List<Product>>` | Isi katalog. |
| `observeCart()` | `Flow<Cart>` | Isi keranjang. |
| `suspend addToCart(productId: ProductId, quantity: Int = 1)` | `PosResult<Cart>` | Tambah dengan validasi stok. |
| `suspend updateCartQuantity(productId: ProductId, quantity: Int)` | `PosResult<Cart>` | Jumlah ≤ 0 menghapus baris. |
| `suspend removeFromCart(productId: ProductId)` | `PosResult<Cart>` | Hapus baris. |
| `suspend clearCart()` | `Unit` | Kosongkan keranjang. |
| `processPayment()` | `Flow<PaymentState>` | Tagih total keranjang. |
| `suspend cancelPayment()` | `Unit` | Batalkan pembayaran yang menunggu tap. |
| `observePaymentState()` | `Flow<PaymentState>` | Status pembayaran terkini. |
| `observeTransactions(limit: Int = 20)` | `Flow<List<Transaction>>` | Riwayat transaksi. |
| `close()` | `Unit` | Lepaskan sumber daya. |

Lihat panduan lengkap di [Pembayaran kartu](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/card-payment).
