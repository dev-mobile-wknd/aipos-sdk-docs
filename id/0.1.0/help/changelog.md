# Catatan rilis

Semua artifact `com.weekendinc.aipos:*` dirilis bersamaan pada satu nomor versi.

## 0.1.0

Rilis pertama.

### Fitur

- **Penyaran produk** — `AdvisorClient` dan `AIPosSDK.pushTranscript` / `observeSuggestions`.
- **Agent kasir** — `AIPosSDK.sendMessage` dengan aksi pencarian produk, keranjang, pembayaran
  kartu, struk, dan riwayat transaksi.
- **Keranjang** dengan validasi stok dan penyimpanan lokal.
- **Pembayaran kartu contactless** di Android lewat MineSec Headless SDK 1.3, dengan simulator
  bawaan.
- **Pembayaran online** — Virtual Account, QRIS, dan kartu lewat halaman pembayaran di WebView,
  dengan pembacaan status otomatis (`attachPaymentPageProbe`), login token otomatis, dan
  pemetaan kode status yang dapat disetel.
- **Kotlin Multiplatform** — Android dan iOS (`iosArm64`, `iosSimulatorArm64`, `iosX64`), dengan
  fungsi jembatan Swift.

### Batasan yang diketahui

- Pembayaran kartu contactless belum didukung di iOS.
- `OnlinePaymentClient` belum menyediakan fungsi keranjang; gunakan `AIPosSDK` untuk pembayaran
  online.
- Status lunas pembayaran online belum diverifikasi ke server oleh SDK.
- SDK memanggil `POST /v1/transactions/simulate-paid` sekitar tiga detik setelah status
  `Pending` bila `baseUrl` terisi.
- Kode status penyedia untuk gagal, kedaluwarsa, dan batal belum terverifikasi di lapangan;
  kode yang tidak dikenal tampil sebagai `WebPaymentStatus.Unknown`.
