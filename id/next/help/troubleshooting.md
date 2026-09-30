# Pemecahan masalah

## Instalasi dan build

### `Could not find com.theminesec.sdk:headless...`

Repository MineSec belum terdaftar atau kredensialnya salah.

1. Pastikan `MINESEC_REGISTRY_LOGIN` dan `MINESEC_REGISTRY_TOKEN` ada di
   `~/.gradle/gradle.properties`.
2. Pastikan blok `maven { url = uri("https://maven.pkg.github.com/theminesec/ms-registry-client") }`
   ada di `settings.gradle.kts` — lihat [Instalasi](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/installation#repository-minesec-pembayaran-kartu).
3. Token GitHub Packages yang kedaluwarsa juga menghasilkan pesan ini. Minta token baru ke MineSec.

### `Dependency ... requires compileSdk 36`

Naikkan `compileSdk` modul aplikasi ke 36.

### `Manifest merger failed: uses-sdk:minSdkVersion 24 cannot be smaller than version 26`

SDK membutuhkan `minSdk = 26`.

### Crash `JNI DETECTED ERROR ... SimpleLoggerFactory`

Ada dua backend SLF4J di classpath: `slf4j-simple` dan `logback-android` (dibawa MineSec).
Kode native MineSec mewajibkan logback. Cari siapa yang membawa `slf4j-simple`:

```bash
./gradlew :app:dependencies --configuration releaseRuntimeClasspath | grep slf4j-simple
```

Lalu buang dari aplikasi Anda:

```kotlin
configurations.configureEach {
    exclude(group = "org.slf4j", module = "slf4j-simple")
}
```

### Log `No SLF4J providers were found`

Bukan error. Aplikasi yang tidak memakai MineSec tidak punya backend logging SLF4J, jadi log
internal tidak ditampilkan. Pasang backend sendiri bila perlu, misalnya
`com.github.tony19:logback-android`.

### Link iOS gagal dengan simbol `std::` tidak ditemukan

Tambahkan `-lc++` ke `OTHER_LDFLAGS`. Lihat [Integrasi iOS](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/platform/ios#b-swift-package-manager-untuk-aplikasi-swift).

## Inisialisasi

### `IllegalArgumentException` saat `build()`

Baca pesannya — setiap pesan menyebut metode builder yang belum dipanggil. Daftar lengkap
ada di [Hasil & error](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/reference/results-and-errors#builder).

### Aplikasi menggantung saat menambah ke keranjang

`ProductCatalogSource` Anda tidak pernah memancarkan nilai. Flow wajib memancarkan nilai
pertama segera — lihat [Katalog produk](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/catalog#pilihan-2-implementasi-sendiri).

## Pembayaran kartu

| Gejala | Penyebab | Solusi |
|---|---|---|
| `PaymentActivityBridge belum di-attach` | `attachActivity` tidak dipanggil, atau dipanggil di `onResume` | Panggil `AiPosAndroid.attachActivity(this)` di `onCreate`, sebelum `setContent` |
| Pembayaran selalu sukses dalam ±3 detik, pesan `[SIMULASI]` | Artifact yang terpasang adalah varian simulator | Pakai artifact yang dibangun dengan MineSec |
| `Profile ID MineSec belum diatur` | `MerchantInfo.profileId` kosong | Isi dari dashboard MineSec |
| `HeadlessActivity must be declared with android:launchMode=singleTask` | Manifest aplikasi menimpa deklarasi Activity SDK | Hapus `tools:replace` untuk `AIPosHeadlessActivity` |
| `initPlatform` gagal soal lisensi | File lisensi tidak ada di `src/main/assets/` atau namanya berbeda | Periksa nama yang dioper ke `licenseName` |
| `Pembayaran contactless belum didukung di iOS` | Dipanggil di iOS | Pakai pembayaran online di iOS |

## Pembayaran online

Saring log terlebih dahulu — banyak masalah langsung terlihat dari sana:

```bash
adb logcat | grep AiPos/online-payment
```

| Gejala | Penyebab | Solusi |
|---|---|---|
| Status diam di `ChoosingMethod` | `attachPaymentPageProbe` belum dipanggil | Panggil `sdk.attachPaymentPageProbe(webView.paymentPageProbe())` setelah WebView dibuat |
| Log: *"tidak ada satu pun respons halaman tersadap"* | Probe terpasang ke WebView yang salah, atau halaman memakai Service Worker / iframe lintas-origin / WebSocket | Pastikan probe dari WebView yang sama yang memuat `paymentUrl` |
| Payload tersadap tetapi status `Unknown(rawStatus = ...)` | Penyedia memakai kode status baru | Tambahkan kodenya ke `statusJsonMapping` — lihat [Pembayaran online](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/online-payment#bila-penyedia-mengubah-kode-status) |
| Endpoint status terlihat di network tab tetapi diabaikan | Alamatnya tidak memuat penggal yang dikenal | Tambahkan penggalnya ke `statusJsonMapping.statusUrlFragments` |
| Tidak ada permintaan ke `/v1/payment-links` | `baseUrl` kosong → mode purwarupa | Isi `baseUrl`; log saat SDK dirakit menyebut mode yang aktif |
| *"Kredensial login pembayaran belum diisi"* | `email` atau `password` kosong | Isi keduanya |
| *"Login pembayaran ditolak server"* | Email/kata sandi salah, atau `baseUrl` salah sehingga `/v1/login` menjawab 404 | Periksa email yang tercetak di log mode, lalu kredensialnya |
| *"Akses pembayaran ditolak walau sudah login ulang"* | Token sudah segar; masalahnya di `apiKey` atau izin akun | Periksa `apiKey` dan peran akun di backend |
| Halaman putih/kosong | JavaScript WebView mati | Pakai `AiposPaymentWebView`, atau panggil `prepareForAiposPayment()` |
| Halaman gagal dimuat, alamat berisi `localhost` | Backend menjawab alamat loopback | Isi `loopbackUrlReplacement` (beta), atau perbaiki `public_url` di backend (produksi) |
| Halaman termuat ulang setiap status berubah | WebView dibuat ulang tiap rekomposisi | Bungkus pembuatan WebView dalam `remember` |
| Halaman tidak bisa digulir di bottom sheet | Gesture sheet menelan sentuhan | Pakai `AiposPaymentWebView` |
| `startOnlinePayment` gagal *"Keranjang masih kosong"* padahal pakai `OnlinePaymentClient` | Keranjang belum diisi sebelum sesi dibuka | Panggil `addToCart(...)` dulu — tersedia di `OnlinePaymentClient` sejak 0.1.1 |
| Di iOS, perpindahan halaman tidak terdeteksi | `WKWebView` tidak memanggil delegate untuk `history.pushState` | Tambahkan KVO pada `webView.url` |

## Fitur AI

| Gejala | Penyebab | Solusi |
|---|---|---|
| Saran tidak pernah muncul | `pushTranscript` tidak pernah dipanggil, atau hanya `setInterimTranscript` | Setor kalimat final lewat `pushTranscript` |
| `observeAdvisorError()` berisi pesan 401 | Kunci OpenAI atau token proxy salah | Periksa `llmApiKey` / proxy |
| 404 dari proxy | `llmProxyBaseUrl` diakhiri `/v1` | Hapus `/v1` |
| Saran selalu kosong padahal ada produk cocok | Model menyebut produk yang tidak persis ada di katalog, lalu dibuang | Lengkapi `name` dan `description` produk |
| Saran mengarah ke pelanggan sebelumnya | Transkrip lama belum dibersihkan | Panggil `clearAdvisor()` atau `resetSession()` saat pelanggan berganti |
| Di iOS, saran tidak pernah dianalisis | Pengenal suara tidak pernah memberi hasil final | Tentukan akhir kalimat dengan jeda hening — lihat [Speech-to-text](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guides/speech-to-text#menentukan-akhir-kalimat) |
