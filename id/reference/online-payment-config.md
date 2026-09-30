# OnlinePaymentConfig

```kotlin
data class OnlinePaymentConfig(...)
```

Paket: `com.weekendinc.aipos.payment.online`. Dioper ke `AIPosSDK.Builder.onlinePayment(...)`
atau `OnlinePaymentClient.Builder.config(...)`.

Semua parameter punya nilai bawaan. Selalu gunakan **argumen bernama**.

## Koneksi dan autentikasi

| Parameter | Tipe | Bawaan | Keterangan |
|---|---|---|---|
| `baseUrl` | `String` | `DEFAULT_BASE_URL` (backend beta) | Alamat API tanpa `/` di akhir dan tanpa `/v1`. **String kosong = mode purwarupa** (tanpa HTTP). |
| `apiKey` | `String` | `""` | Kunci perangkat, dikirim sebagai header `X-API-Key`. Satu-satunya yang tidak punya bawaan bermakna. |
| `email` | `String` | `DEFAULT_EMAIL` (akun beta) | Akun yang ditukar jadi token lewat `POST {baseUrl}/v1/login`. |
| `password` | `String` | `DEFAULT_PASSWORD` (akun beta) | Pasangan `email`. Ikut terbundel di aplikasi — lihat [Siap produksi](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guides/production-checklist#keamanan-kunci). |
| `requestTimeoutMillis` | `Long` | `10_000` | Batas waktu satu permintaan HTTP. |
| `loopbackUrlReplacement` | `String` | `DEFAULT_LOOPBACK_URL_REPLACEMENT` (halaman purwarupa) | Skema+host pengganti bila API menjawab dengan alamat `localhost`. **Isi `""` di produksi.** |

## Tagihan

| Parameter | Tipe | Bawaan | Keterangan |
|---|---|---|---|
| `linkExpiry` | `kotlin.time.Duration` | `24.hours` | Umur tautan pembayaran sejak dibuat. |
| `adminFee` | `Money` | `Money.ZERO` | Biaya admin di luar harga barang. Nol berarti tidak dikirim. |
| `allowedPaymentMethods` | `Set<OnlinePaymentChannel>` | `OnlinePaymentChannel.DEFAULT` (VA, QRIS, kartu) | Kanal yang boleh dipilih pelanggan, ditampilkan sesuai urutan. |

## Tampilan halaman

| Parameter | Tipe | Bawaan | Keterangan |
|---|---|---|---|
| `appearance` | `PaymentLinkAppearance` | kosong | Logo dan warna halaman. Field kosong tidak dikirim. |
| `customerData` | `CustomerDataPolicy` | semua mati | Data pelanggan yang diminta di halaman. |
| `hiddenButtonLabels` | `List<String>` | `["back to merchant", "kembali ke merchant"]` | Tombol/tautan yang **memuat** teks ini disembunyikan (tidak peka huruf besar-kecil). List kosong mematikan fitur. |

### `PaymentLinkAppearance`

| Field | Tipe | Contoh |
|---|---|---|
| `logoUrl` | `String` | `"https://tokoanda.co.id/logo.png"` — harus bisa diakses publik |
| `backgroundColor` | `String` | `"#9e086c"` |
| `buttonColor` | `String` | `"#273395"` |
| `textMode` | `PaymentLinkTextMode?` | `LIGHT`, `DARK`, atau `null` (bawaan penyedia) |

### `CustomerDataPolicy`

Berisi `name`, `email`, `phone`, dan `address`, masing-masing bertipe
`CollectCustomerField(enabled: Boolean = false, required: Boolean = false)`.

```kotlin
customerData = CustomerDataPolicy(
    email = CollectCustomerField(enabled = true, required = true),
)
```

## Pemantauan status

| Parameter | Tipe | Bawaan | Keterangan |
|---|---|---|---|
| `statusJsonMapping` | `PaymentStatusJsonMapping` | kode penyedia saat ini | Cara menerjemahkan jawaban status halaman. |
| `pagePollIntervalMillis` | `Long` | `1_000` | Jarak antar-pembacaan halaman pembayaran. |
| `captureWarningDelayMillis` | `Long` | `15_000` | Setelah selama ini tanpa satu pun respons terbaca, SDK menulis peringatan ke log. |

### `PaymentStatusJsonMapping`

| Field | Bawaan |
|---|---|
| `statusFieldNames` | `status`, `transaction_status`, `payment_status`, `state` |
| `transactionIdFieldNames` | `transaction`, `transaction_id`, `transactionId`, `id` |
| `choosingMethodValues` | `active` |
| `pendingValues` | `PNDNG`, `pending` |
| `successValues` | `SETLD`, `settled`, `success`, `paid` |
| `failedValues` | `FAILD`, `EXPRD`, `RJCTD`, `DECLN`, `failed`, `failure`, `error`, `rejected`, `expired` |
| `cancelledValues` | `CNCLD`, `cancelled`, `canceled` |
| `statusUrlFragments` | `payment-link`, `transaction` |
| `methodFieldNames`, `vaNumberFieldNames`, `channelValues` | Dipakai untuk pelunasan tiruan saja |

Setiap nilai bawaan tersedia sebagai konstanta `DEFAULT_*` di `companion object`, sehingga
Anda bisa menambah tanpa menulis ulang:

```kotlin
PaymentStatusJsonMapping(
    successValues = PaymentStatusJsonMapping.DEFAULT_SUCCESS_VALUES + "CMPLT",
)
```

Pencocokan nilai tidak peka huruf besar-kecil.

## Properti turunan

| Properti | Keterangan |
|---|---|
| `isSimulated` | `true` bila `baseUrl` kosong (mode purwarupa). |
| `hasLoginCredentials` | `true` bila `email` dan `password` sama-sama terisi. |

## Contoh produksi

```kotlin
OnlinePaymentConfig(
    baseUrl = "https://pay-api.tokoanda.co.id",
    apiKey = kunciPerangkat,
    email = akun.email,
    password = akun.password,
    loopbackUrlReplacement = "",
    allowedPaymentMethods = setOf(OnlinePaymentChannel.VIRTUAL_ACCOUNT, OnlinePaymentChannel.QRIS),
    appearance = PaymentLinkAppearance(logoUrl = "https://tokoanda.co.id/logo.png"),
)
```

Dari Swift, pakai [`onlinePaymentConfigOf`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/reference/swift-bridge#onlinepaymentconfigof).
