# Konfigurasi

Semua konfigurasi diberikan lewat builder saat SDK dirakit. `build()` memvalidasi
konfigurasi wajib dan melempar `IllegalArgumentException` bila ada yang kurang — kesalahan
wiring ketahuan saat aplikasi dibuka, bukan saat pelanggan sudah menunggu di kasir.

## Ringkasan builder

| Metode | `AIPosSDK` | `AdvisorClient` | `PaymentClient` | `OnlinePaymentClient` |
|---|:---:|:---:|:---:|:---:|
| `.merchantInfo(...)` | wajib | — | wajib | wajib |
| `.productCatalog(...)` | wajib | wajib | wajib | wajib |
| `.llmApiKey(...)` / `.llmProxyBaseUrl(...)` | salah satu wajib | salah satu wajib | — | — |
| `.onlinePayment(...)` / `.config(...)` | opsional | — | — | opsional |
| `.android(context)` / `.ios()` | wajib | — | wajib | wajib |

## Identitas merchant

`MerchantInfo` dicetak di struk dan dikirim ke gateway pembayaran.

```kotlin
val merchant = MerchantInfo(
    id = "merchant_001",                     // your internal id
    name = "Jaya Gadget Store",              // printed on the receipt
    address = "Jl. Sudirman No. 1, Jakarta",
    terminalId = "TID001",                   // Terminal ID from the acquirer
    mid = "MID123456",                       // Merchant ID from the acquirer
    profileId = "prof_xxxxxxxx",             // your MineSec profile
)
```

| Field | Keterangan |
|---|---|
| `id` | Identitas merchant di sistem Anda. |
| `name` | Nama toko yang tercetak di struk. |
| `address` | Alamat toko. |
| `terminalId` | Terminal ID (TID) dari acquirer. |
| `mid` | Merchant ID (MID) dari acquirer. |
| `profileId` | Profil terminal MineSec dari dashboard MineSec. Bawaannya `MerchantInfo.DEFAULT_PROFILE_ID`, profil **lingkungan uji**. Merchant produksi wajib mengisi profilnya sendiri. |

## Model bahasa

Fitur AI (penyaran produk dan agent kasir) membutuhkan akses ke model bahasa. Pilih salah
satu cara:

**Pengembangan: API key langsung**

```kotlin
AIPosSDK.Builder()
    .llmApiKey(BuildConfig.LLM_API_KEY)
    // ...
```

**Produksi: lewat proxy**

```kotlin
AIPosSDK.Builder()
    .llmProxyBaseUrl("https://api.tokoanda.com/llm")
    // ...
```

| Situasi | Yang dipakai SDK |
|---|---|
| Hanya `llmApiKey` diisi | Memanggil OpenAI langsung dengan kunci tersebut. |
| Hanya `llmProxyBaseUrl` diisi | Memanggil proxy tanpa kunci. |
| Keduanya diisi | Memanggil **proxy**, dan `llmApiKey` dikirim sebagai token autentikasi ke proxy. |
| Keduanya kosong | `build()` gagal: *"Salah satu dari llmApiKey(...) atau llmProxyBaseUrl(...) wajib diisi"*. |

:::danger[API key di aplikasi bisa diekstrak]
Siapa pun yang mengunduh APK Anda bisa membongkar kunci yang dibundel di dalamnya. Pakai
`llmApiKey` hanya untuk pengembangan dan demo. Untuk produksi, lihat
[Proxy LLM](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guides/llm-proxy).
:::

:::warning[Tanpa `/v1`]
Isi `llmProxyBaseUrl` **tanpa** akhiran `/v1`. SDK menambahkan `v1/chat/completions`
sendiri, jadi `https://api.tokoanda.com/llm/v1` akan menghasilkan URL ganda dan berujung 404.
:::

## Katalog produk

Wajib untuk semua titik masuk. Lihat [Katalog produk](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/catalog).

```kotlin
.productCatalog(MutableProductCatalog(storeProducts))
```

## Pembayaran online

Opsional. Tanpa pemanggilan ini, SDK memakai `OnlinePaymentConfig()` bawaan yang menunjuk
**lingkungan beta** — cocok untuk mencoba alur, bukan untuk transaksi sungguhan.

```kotlin
AIPosSDK.Builder()
    .onlinePayment(
        OnlinePaymentConfig(
            baseUrl = "https://pay-api.tokoanda.co.id",   // without /v1
            apiKey = BuildConfig.POS_BACKEND_API_KEY,
            email = "cashier@tokoanda.co.id",
            password = passwordFromSecureStorage,
            loopbackUrlReplacement = "",                  // disable in production
        )
    )
```

:::warning[Selalu pakai argumen bernama]
`OnlinePaymentConfig` punya lima belas parameter. Memanggilnya secara posisional mudah
tertukar tanpa error kompilasi — misalnya kata sandi masuk sebagai email.
:::

Tiga mode yang bisa Anda pilih:

| Mode | Cara mengaktifkan | Perilaku |
|---|---|---|
| **Purwarupa** | `OnlinePaymentConfig(baseUrl = "")` | Tidak ada permintaan HTTP sama sekali. SDK memakai halaman pembayaran contoh. Untuk mengembangkan UI. |
| **Beta** | Tidak memanggil `.onlinePayment(...)`, atau hanya mengisi `apiKey` | Backend beta bersama milik SDK. Untuk uji integrasi. |
| **Produksi** | Isi `baseUrl`, `apiKey`, `email`, `password` milik Anda | Backend merchant Anda sendiri. |

Daftar lengkap parameter ada di [OnlinePaymentConfig](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/reference/online-payment-config).

## Menyimpan rahasia

Jangan menulis kunci langsung di kode. Pola sederhana untuk Android:

```properties
# local.properties — gitignored by default
LLM_API_KEY=sk-proj-...
POS_BACKEND_API_KEY=...
```

```kotlin title="app/build.gradle.kts"
import java.util.Properties

val localProps = Properties().apply {
    rootProject.file("local.properties").takeIf { it.exists() }?.inputStream()?.use(::load)
}

android {
    buildFeatures { buildConfig = true }
    defaultConfig {
        buildConfigField("String", "LLM_API_KEY", "\"${localProps.getProperty("LLM_API_KEY", "")}\"")
        buildConfigField("String", "POS_BACKEND_API_KEY", "\"${localProps.getProperty("POS_BACKEND_API_KEY", "")}\"")
    }
}
```

Cara ini menjauhkan kunci dari Git, **tetapi kunci tetap ikut terbundel di APK**. Untuk
produksi, pindahkan kunci ke server — lihat [Siap produksi](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guides/production-checklist).

## Pesan error dari `build()`

| Pesan | Penyebab |
|---|---|
| `MerchantInfo wajib diisi — panggil merchantInfo(...)` | `.merchantInfo(...)` belum dipanggil. |
| `Sumber katalog wajib diisi — panggil productCatalog(...)` | `.productCatalog(...)` belum dipanggil. |
| `Platform belum diatur — panggil .android(context) di Android atau .ios() di iOS` | Lupa memanggil `.android(context)` atau `.ios()`. |
| `Salah satu dari llmApiKey(...) atau llmProxyBaseUrl(...) wajib diisi` | Kedua jalur model bahasa kosong. |
