# Hasil & error

## `PosResult`

```kotlin
sealed class PosResult<out T> {
    data class Success<out T>(val data: T) : PosResult<T>()
    data class Failure(val message: String, val cause: Throwable? = null) : PosResult<Nothing>()
}
```

Paket `com.weekendinc.aipos.domain.common`. Operasi yang bisa gagal mengembalikan
`PosResult` alih-alih melempar exception, sehingga kemungkinan gagal terlihat di signature.

| Anggota | Keterangan |
|---|---|
| `isSuccess` | `true` bila `Success`. |
| `getOrNull()` | Data bila sukses, `null` bila gagal. |
| `map { }` | Ubah data sukses; kegagalan diteruskan apa adanya. |
| `onSuccess { data -> }` | Jalankan bila sukses. Mengembalikan hasil yang sama. |
| `onFailure { failure -> }` | Jalankan bila gagal. Mengembalikan hasil yang sama. |

`Failure.message` **aman ditampilkan** ke kasir. `Failure.cause` berisi detail teknis.

```kotlin
sdk.addToCart(id)
    .onSuccess { cart -> updateBadge(cart.itemCount) }
    .onFailure { failure ->
        snackbar(failure.message)
        Log.w("POS", "addToCart failed", failure.cause)
    }
```

## `OnlinePaymentErrorCode`

Paket `com.weekendinc.aipos.domain.error`. Tersedia pada kegagalan pembayaran online lewat
extension `PosResult.Failure.onlineErrorCode` (`null` untuk sebab lain).

| Kode | Arti | Boleh dicoba ulang? |
|---|---|:---:|
| `NETWORK` | Jaringan putus atau backend tidak terjangkau | ✅ |
| `UNAUTHORIZED` | Kredensial kosong atau ditolak, termasuk setelah login ulang otomatis | ❌ perbaiki konfigurasi |
| `INVALID_REQUEST` | Data permintaan tidak lolos validasi backend | ❌ |
| `DUPLICATE_ORDER` | `orderId` sudah dipakai untuk tagihan berbeda | ❌ |
| `EXPIRED` | Tagihan melewati batas waktu | buat sesi baru |
| `PROVIDER_ERROR` | Penyedia pembayaran menolak atau bermasalah | ✅ nanti |
| `UNKNOWN` | Sebab tidak dikenali | — |

```kotlin
import com.weekendinc.aipos.domain.error.OnlinePaymentErrorCode
import com.weekendinc.aipos.domain.error.onlineErrorCode

val result = sdk.startOnlinePayment()
if (result is PosResult.Failure && result.onlineErrorCode == OnlinePaymentErrorCode.NETWORK) {
    showRetryButton()
}
```

## Pesan kegagalan umum

### Keranjang

| Pesan | Dari |
|---|---|
| `Jumlah harus lebih dari 0` | `addToCart` |
| `Produk dengan id '<id>' tidak ditemukan` | `addToCart` |
| `<nama> sedang habis` | `addToCart` |
| `Stok <nama> tidak cukup. Tersedia <n>, diminta <m>` | `addToCart`, `updateCartQuantity` |
| `Produk tidak ada di keranjang` | `updateCartQuantity`, `removeFromCart` |

### Pembayaran

| Pesan | Dari |
|---|---|
| `Keranjang masih kosong` | `processPayment` (sebagai `PaymentState.Failed`), `startOnlinePayment` |
| `Total belanja harus lebih dari Rp 0` | `processPayment`, `startOnlinePayment` |
| `Total belanja harus dalam Rupiah bulat` | `startOnlinePayment` |
| `Pembayaran contactless belum didukung di iOS...` | `processPayment` di iOS |

### Builder

| Pesan | Penyebab |
|---|---|
| `MerchantInfo wajib diisi — panggil merchantInfo(...)` | `merchantInfo` belum dipanggil |
| `Sumber katalog wajib diisi — panggil productCatalog(...)` | `productCatalog` belum dipanggil |
| `Platform belum diatur — panggil .android(context) di Android atau .ios() di iOS` | Platform belum diatur |
| `Salah satu dari llmApiKey(...) atau llmProxyBaseUrl(...) wajib diisi` | Kedua jalur model bahasa kosong |
