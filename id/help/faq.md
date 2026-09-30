# FAQ

## Umum

### Apakah SDK menyediakan tampilan siap pakai?

Tidak. SDK hanya menyediakan logika, data, dan alur transaksi. Seluruh tampilan milik aplikasi
Anda, sehingga desainnya bisa mengikuti merek toko.

### Apakah saya harus memakai semua fitur?

Tidak. Pilih artifact sesuai kebutuhan — lihat [Memilih artifact](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/choosing-artifacts).

### Apakah SDK bisa berjalan tanpa internet?

Keranjang dan pembayaran kartu tidak membutuhkan internet dari sisi SDK (terminal MineSec
punya kebutuhan jaringannya sendiri). Penyaran produk, agent kasir, dan pembayaran online
membutuhkan internet.

### Apakah SDK bentrok dengan Koin atau Hilt di aplikasi saya?

Tidak. SDK memakai container Koin miliknya sendiri, bukan `startKoin` global.

## Data

### Di mana data keranjang dan transaksi disimpan?

Di penyimpanan lokal perangkat (`SharedPreferences` di Android, `NSUserDefaults` di iOS).
Keranjang bertahan saat aplikasi ditutup. SDK tidak mengirim transaksi ke server mana pun —
kirim sendiri ke backend Anda bila perlu.

### Apakah SDK mengurangi stok setelah transaksi?

Tidak. SDK hanya memvalidasi terhadap angka `stock` yang Anda berikan. Lihat
[Katalog produk](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/catalog#stok-tetap-tanggung-jawab-anda).

### Mata uang apa yang didukung?

Rupiah. `Money.format()` menghasilkan `"Rp 25.000"`, dan pembayaran online hanya menerima
nominal Rupiah bulat.

## Fitur AI

### Model bahasa apa yang dipakai?

OpenAI GPT-5.4 mini, lewat API OpenAI langsung atau proxy yang kompatibel.

### Data apa yang dikirim ke penyedia model bahasa?

Untuk penyaran produk: transkrip percakapan dan data katalog (nama, kategori, harga,
deskripsi, stok). Untuk agent kasir: pesan kasir, hasil pencarian produk, dan isi keranjang.
Tidak ada data kartu pembayaran yang dikirim.

### Bahasa apa yang dipahami?

Bahasa Indonesia, termasuk campuran Indonesia–Inggris yang lazim di toko. Jawaban dan alasan
saran ditulis dalam Bahasa Indonesia.

### Apakah SDK merekam suara?

Tidak. SDK tidak pernah mengakses mikrofon. Aplikasi Anda yang merekam dan mengubahnya
jadi teks.

## Pembayaran

### Kartu apa saja yang diterima terminal?

Visa, Mastercard, American Express, JCB, Maestro, dan UnionPay — bergantung pada konfigurasi
acquiring di profil MineSec Anda.

### Bisakah iPhone menerima pembayaran kartu?

Belum. Lihat [Pengenalan](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/introduction#dukungan-platform). Gunakan pembayaran online di iOS.

### Bagaimana cara mencoba pembayaran tanpa uang sungguhan?

- **Kartu:** varian simulator menyetujui setiap transaksi.
- **Online:** mode purwarupa (`baseUrl = ""`) atau lingkungan beta bawaan.

### Kenapa saya harus memverifikasi pembayaran online ke backend?

Status lunas dibaca dari halaman pembayaran di perangkat kasir, bukan dari server. Lihat
[Pembayaran online](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/online-payment#verifikasi-pembayaran-di-backend).

### Bisakah kasir memilih kanal (VA atau QRIS) sebelum halaman dibuka?

Tidak perlu. Pelanggan memilih sendiri di halaman pembayaran. Anda hanya bisa membatasi kanal
yang tersedia lewat `allowedPaymentMethods`.
