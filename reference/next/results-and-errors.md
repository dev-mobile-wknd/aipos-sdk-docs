# Results & errors

## `PosResult`

```kotlin
sealed class PosResult<out T> {
    data class Success<out T>(val data: T) : PosResult<T>()
    data class Failure(val message: String, val cause: Throwable? = null) : PosResult<Nothing>()
}
```

Package `com.weekendinc.aipos.domain.common`. Operations that can fail return a
`PosResult` instead of throwing an exception, so the chance of failure is visible
in the signature.

| Member | Description |
|---|---|
| `isSuccess` | `true` when `Success`. |
| `getOrNull()` | The data on success, `null` on failure. |
| `map { }` | Transform the success data; a failure is passed through unchanged. |
| `onSuccess { data -> }` | Runs on success. Returns the same result. |
| `onFailure { failure -> }` | Runs on failure. Returns the same result. |

`Failure.message` is **safe to display** to the cashier. `Failure.cause` holds
technical detail.

```kotlin
sdk.addToCart(id)
    .onSuccess { cart -> updateBadge(cart.itemCount) }
    .onFailure { failure ->
        snackbar(failure.message)
        Log.w("POS", "addToCart failed", failure.cause)
    }
```

## `OnlinePaymentErrorCode`

Package `com.weekendinc.aipos.domain.error`. Available on online-payment
failures via the `PosResult.Failure.onlineErrorCode` extension (`null` for other
causes).

| Code | Meaning | Retryable? |
|---|---|:---:|
| `NETWORK` | Network down or backend unreachable | ✅ |
| `UNAUTHORIZED` | Credentials empty or rejected, even after an automatic re-login | ❌ fix the configuration |
| `INVALID_REQUEST` | Request data didn't pass backend validation | ❌ |
| `DUPLICATE_ORDER` | `orderId` was already used for a different charge | ❌ |
| `EXPIRED` | The charge passed its deadline | start a new session |
| `PROVIDER_ERROR` | The payment provider declined or had an issue | ✅ later |
| `UNKNOWN` | Unrecognized cause | — |

```kotlin
import com.weekendinc.aipos.domain.error.OnlinePaymentErrorCode
import com.weekendinc.aipos.domain.error.onlineErrorCode

val result = sdk.startOnlinePayment()
if (result is PosResult.Failure && result.onlineErrorCode == OnlinePaymentErrorCode.NETWORK) {
    showRetryButton()
}
```

## Common failure messages

The SDK's failure messages are in Indonesian, matching its primary market.

### Cart

| Message | From |
|---|---|
| `Jumlah harus lebih dari 0` (quantity must be greater than 0) | `addToCart` |
| `Produk dengan id '<id>' tidak ditemukan` (product with id `<id>` not found) | `addToCart` |
| `<name> sedang habis` (`<name>` is out of stock) | `addToCart` |
| `Stok <name> tidak cukup. Tersedia <n>, diminta <m>` (not enough stock for `<name>`. `<n>` available, `<m>` requested) | `addToCart`, `updateCartQuantity` |
| `Produk tidak ada di keranjang` (product not in the cart) | `updateCartQuantity`, `removeFromCart` |

### Payment

| Message | From |
|---|---|
| `Keranjang masih kosong` (cart is still empty) | `processPayment` (as `PaymentState.Failed`), `startOnlinePayment` |
| `Total belanja harus lebih dari Rp 0` (the purchase total must be greater than Rp 0) | `processPayment`, `startOnlinePayment` |
| `Total belanja harus dalam Rupiah bulat` (the purchase total must be a whole Rupiah amount) | `startOnlinePayment` |
| `Pembayaran contactless belum didukung di iOS...` (contactless payment isn't supported on iOS yet...) | `processPayment` on iOS |

### Builder

| Message | Cause |
|---|---|
| `MerchantInfo wajib diisi — panggil merchantInfo(...)` (MerchantInfo is required — call merchantInfo(...)) | `merchantInfo` hasn't been called |
| `Sumber katalog wajib diisi — panggil productCatalog(...)` (a catalog source is required — call productCatalog(...)) | `productCatalog` hasn't been called |
| `Platform belum diatur — panggil .android(context) di Android atau .ios() di iOS` (platform not set — call .android(context) on Android or .ios() on iOS) | The platform hasn't been set |
| `Salah satu dari llmApiKey(...) atau llmProxyBaseUrl(...) wajib diisi` (one of llmApiKey(...) or llmProxyBaseUrl(...) is required) | Both language-model paths are empty |
