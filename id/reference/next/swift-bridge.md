# Jembatan Swift

Sebagian API Kotlin tidak terbawa utuh ke Swift. SDK menyediakan fungsi pembantu di source
set iOS untuk menutup celah tersebut. Nama kelas di Swift mengikuti nama berkas Kotlin-nya
(`NamaBerkasKt`).

## `FlowSubscriptionKt.subscribe`

```swift
func subscribe(_ flow: Kotlinx_coroutines_coreFlow, onEach: @escaping (Any?) -> Void) -> FlowSubscription
```

Amati `Flow` dari Swift. Setiap nilai dikirim **di ulir utama**.

```swift
let subscription = FlowSubscriptionKt.subscribe(sdk.observeCart()) { value in
    guard let cart = value as? Cart else { return }
    self.cart = cart
}
subscription.cancel()   // required when the screen is closed
```

`FlowSubscription` punya `isActive: Bool` dan `cancel()` (aman dipanggil berkali-kali).

## `IosDomainFactoryKt`

### `productOf`

```swift
IosDomainFactoryKt.productOf(
    id: String, name: String, priceRupiah: Int64,
    barcode: String, category: String, stock: Int32,
    description: String, imageUrl: String
) -> Product
```

Rakit `Product` dari tipe biasa. Harga dalam **Rupiah penuh**.

### `formatMoneyCents`

```swift
IosDomainFactoryKt.formatMoneyCents(cents: Int64) -> String
```

Format nilai uang dari properti seperti `cart.total` atau `product.price` (bertipe `Int64`
berisi sen) menjadi `"Rp 13.999.000"`.

### `productIdValue`

```swift
IosDomainFactoryKt.productIdValue(product: Product) -> String
```

Baca id produk sebagai `String` — misalnya untuk `Identifiable` di SwiftUI. Untuk memanggil
`addToCart(productId:)`, oper `product.id` apa adanya.

## `IosOnlinePaymentConfigFactoryKt`

### `onlinePaymentConfigOf`

```swift
// Device key only; everything else uses the SDK default (beta environment)
IosOnlinePaymentConfigFactoryKt.onlinePaymentConfigOf(apiKey: String) -> OnlinePaymentConfig

// Full
IosOnlinePaymentConfigFactoryKt.onlinePaymentConfigOf(
    baseUrl: String, apiKey: String, email: String, password: String,
    loopbackUrlReplacement: String
) -> OnlinePaymentConfig
```

Parameter lain (`linkExpiry`, `appearance`, dan seterusnya) memakai nilai bawaan. Untuk
mengubahnya, rakit `OnlinePaymentConfig` di sisi Kotlin.

## `IosPaymentPageProbeKt`

### `paymentPageProbe`

```swift
IosPaymentPageProbeKt.paymentPageProbe(_ webView: WKWebView) -> PaymentPageProbe
```

Probe untuk `sdk.attachPaymentPageProbe(probe:)`. Wajib agar status pembayaran online bergerak.

## Tipe yang berubah bentuk

| Kotlin | Swift |
|---|---|
| `Money` | `Int64` (sen) |
| `ProductId` | `Any` |
| `Flow<T>` | `Kotlinx_coroutines_coreFlow` — nilainya `Any?` |
| `PosResult.Success<T>` | `PosResultSuccess<T>` — data di `.data` |
| `PosResult.Failure` | `PosResultFailure` — pesan di `.message` |
| `sealed class` objek (`AdvisorState.Idle`) | Kelas dengan inisialisasi `AdvisorState.Idle()`; bandingkan dengan `is` |
| Konstanta companion (`MerchantInfo.DEFAULT_PROFILE_ID`) | `MerchantInfo.companion.DEFAULT_PROFILE_ID` |
| Fungsi `suspend` | `async throws` |
| Parameter bernilai bawaan | Wajib diisi |
