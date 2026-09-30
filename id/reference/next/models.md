# Model data

Semua model adalah `data class` atau `sealed class` yang *immutable*.

## Nilai dasar

### `Money`

Paket `com.weekendinc.aipos.domain.valueobject`. Uang disimpan sebagai bilangan bulat
**sen** (1/100 Rupiah) untuk menghindari galat pembulatan.

| Anggota | Keterangan |
|---|---|
| `Money.fromRupiah(amount: Long)` | Buat dari Rupiah penuh. `fromRupiah(25_000)` = Rp 25.000. |
| `Money(cents: Long)` | Buat dari sen. `Money(2_500_000)` = Rp 25.000. |
| `Money.ZERO` | Nol. |
| `cents` | Nilai dalam sen. |
| `whole` | Nilai dalam Rupiah penuh. |
| `format()` | `"Rp 25.000"`. Juga dipakai `toString()`. |
| `isNotPositive` | `true` bila nol atau negatif. |
| `+`, `-`, `* Int`, `compareTo` | Operasi aritmetika dan perbandingan. |

### `ProductId`

Paket `com.weekendinc.aipos.domain.valueobject`. Membungkus `String`; tidak boleh kosong.

```kotlin
val id = ProductId("GDG-001")
id.value   // "GDG-001"
```

## Katalog dan keranjang

Paket `com.weekendinc.aipos.domain.entity`.

### `Product`

| Field | Tipe | Bawaan |
|---|---|---|
| `id` | `ProductId` | — |
| `name` | `String` | — |
| `price` | `Money` | — |
| `barcode` | `String` | — |
| `category` | `String` | — |
| `stock` | `Int` | — |
| `description` | `String` | `""` |
| `imageUrl` | `String` | `""` |
| `isAvailable` | `Boolean` (turunan) | `stock > 0` |

### `Cart`

| Anggota | Tipe |
|---|---|
| `items` | `List<CartItem>` |
| `discount` | `Money` |
| `subtotal` | `Money` |
| `total` | `Money` — tidak pernah negatif |
| `itemCount` | `Int` — total kuantitas |
| `isEmpty` | `Boolean` |
| `findItem(productId)` | `CartItem?` |

### `CartItem`

`product: Product`, `quantity: Int` (minimal 1), `subtotal: Money`.

### `MerchantInfo`

| Field | Tipe | Bawaan |
|---|---|---|
| `id` | `String` | — |
| `name` | `String` | — |
| `address` | `String` | — |
| `terminalId` | `String` | — |
| `mid` | `String` | — |
| `profileId` | `String` | `MerchantInfo.DEFAULT_PROFILE_ID` (profil uji) |

## Penyaran produk

Paket `com.weekendinc.aipos.advisor`.

### `ProductSuggestion`

| Field | Tipe | Keterangan |
|---|---|---|
| `product` | `Product` | Produk utuh dari katalog |
| `reason` | `String` | Alasan dalam Bahasa Indonesia |
| `confidence` | `Double` | 0.0 – 1.0 |

### `AdvisorState`

`sealed class` dengan tiga objek: `Idle`, `Thinking`, `Suggesting`.

## Agent kasir

### `AgentMessage`

Paket `com.weekendinc.aipos.agent.model`. Semua subtipe punya `timestamp: Long`.

| Subtipe | Field |
|---|---|
| `UserMessage` | `content: String` |
| `AssistantMessage` | `content: String` |
| `ToolCall` | `toolName: String`, `summary: String` |
| `ErrorMessage` | `message: String` |

## Pembayaran kartu

### `PaymentState`

Paket `com.weekendinc.aipos.domain.model`.

| Subtipe | Field |
|---|---|
| `Idle` | — |
| `WaitingTap` | `message: String`, `timeoutSeconds: Int` |
| `Processing` | `message: String`, `cardScheme: CardScheme` |
| `Success` | `transactionId: String`, `amount: Money`, `paymentMethod: PaymentMethod`, `cardScheme: CardScheme`, `rrn: String?`, `approvalCode: String?`, `posReference: String?` |
| `Failed` | `errorMessage: String`, `responseCode: String?`, `isCancelled: Boolean`, `isTimeout: Boolean` |

`isTerminal: Boolean` — `true` untuk `Success` dan `Failed`.

### `Transaction`

Paket `com.weekendinc.aipos.domain.entity`.

| Field | Tipe |
|---|---|
| `id` | `String` |
| `cart` | `Cart` |
| `amount` | `Money` |
| `paymentMethod` | `PaymentMethod` |
| `status` | `TransactionStatus` |
| `timestamp` | `Long` |
| `rrn` | `String?` |
| `approvalCode` | `String?` |
| `posReference` | `String?` |
| `isApproved` | `Boolean` (turunan) |

### `TransactionStatus`

`PENDING`, `APPROVED`, `DECLINED`, `CANCELLED`, `TIMEOUT`.

### `PaymentMethod`

| Subtipe | Field | `displayName` |
|---|---|---|
| `Contactless` | `scheme: CardScheme`, `maskedPan: String?` | `"VISA **** 4242"` |
| `QRCode` | `issuer: String = "QRIS"` | `"QRIS"` |
| `Cash` | — | `"TUNAI"` |

### `CardScheme`

`VISA`, `MASTERCARD`, `AMEX`, `JCB`, `MAESTRO`, `UNIONPAY`, `UNKNOWN`.

## Pembayaran online

Paket `com.weekendinc.aipos.domain.entity`.

### `WebPaymentSession`

| Field | Tipe |
|---|---|
| `sessionId` | `String` |
| `orderId` | `String` |
| `paymentUrl` | `String` |
| `amount` | `Money` |
| `expiresAtMillis` | `Long?` |

### `WebPaymentStatus`

Semua subtipe punya `transactionId: String?`.

| Subtipe | Field tambahan | `isFinal` |
|---|---|:---:|
| `Idle` | — | |
| `ChoosingMethod` | — | |
| `Pending` | — | |
| `Success` | — | ✅ |
| `Failed` | `rawStatus: String?` | ✅ |
| `Cancelled` | — | ✅ |
| `Unknown` | `rawStatus: String` | |

### `OnlinePaymentChannel`

| Nilai | `code` |
|---|---|
| `VIRTUAL_ACCOUNT` | `"VA"` |
| `QRIS` | `"QR"` |
| `CARD` | `"CARD"` |

`OnlinePaymentChannel.ALL` dan `OnlinePaymentChannel.DEFAULT` — keduanya berisi ketiga kanal
pada versi 0.1.1.
