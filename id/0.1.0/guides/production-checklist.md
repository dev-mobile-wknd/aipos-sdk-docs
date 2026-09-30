# Siap produksi

Periksa daftar ini sebelum aplikasi dipasang di perangkat kasir.

## Keamanan kunci

- [ ] **Kunci OpenAI tidak ada di aplikasi.** Pakai [`llmProxyBaseUrl`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guides/llm-proxy).
- [ ] **Kata sandi backend pembayaran tidak dibundel.** `OnlinePaymentConfig.password` ikut
      terbundel ke APK/IPA dan tidak bisa dicabut per perangkat. Ambil dari server saat
      perangkat diaktivasi dan simpan di Android Keystore / iOS Keychain.
- [ ] **`apiKey` pembayaran berbeda per perangkat**, supaya perangkat yang hilang bisa
      dicabut tanpa mengganggu kasir lain.
- [ ] Kredensial MineSec hanya ada di `~/.gradle/gradle.properties` atau secret CI.

## Pembayaran kartu

- [ ] Transaksi uji di build rilis **tidak** menampilkan pesan `[SIMULASI]` dan `transactionId`
      tidak diawali `SIM-`.
- [ ] Lisensi MineSec produksi ada di `src/main/assets/` dan namanya dioper ke `initPlatform`.
      Jangan memakai `public-test.license`.
- [ ] `MerchantInfo.profileId` diisi profil produksi, bukan `DEFAULT_PROFILE_ID`.
- [ ] `terminalId` dan `mid` sesuai data dari acquirer.
- [ ] `initPlatform` yang gagal ditampilkan ke kasir, bukan hanya dicatat di log.

## Pembayaran online

- [ ] `baseUrl`, `email`, `password`, dan `apiKey` menunjuk **backend produksi Anda**, bukan
      bawaan beta.
- [ ] `loopbackUrlReplacement = ""` — bawaannya mengarahkan alamat `localhost` ke halaman
      purwarupa.
- [ ] **Status lunas dikonfirmasi ke backend** sebelum barang diserahkan. `WebPaymentStatus.Success`
      dibaca dari perangkat kasir dan belum diverifikasi server.
- [ ] **Endpoint `POST /v1/transactions/simulate-paid` tidak aktif di backend produksi.** SDK
      versi 0.1.0 memanggilnya sekitar tiga detik setelah status `Pending` untuk keperluan
      uji. Bila endpoint itu ada dan menerima panggilan, transaksi bisa tercatat lunas tanpa
      pembayaran sungguhan.
- [ ] `allowedPaymentMethods` sesuai kanal yang benar-benar diterima toko.
- [ ] Kode status penyedia untuk gagal dan kedaluwarsa sudah diuji. Kode yang belum dikenal
      tampil sebagai `Unknown` — pantau log `AiPos/online-payment` di minggu pertama.

## Fitur AI

- [ ] Kebijakan privasi toko menyebut bahwa percakapan dikirim ke penyedia model bahasa.
- [ ] Proxy LLM punya batas laju dan pemantauan biaya.
- [ ] `Product.description` terisi — kualitas saran sangat bergantung padanya.

## Data dan stok

- [ ] Stok dikurangi di sistem Anda setelah transaksi lunas. SDK tidak pernah menguranginya.
- [ ] Transaksi dikirim ke backend Anda. Riwayat di SDK hanya tersimpan di perangkat.
- [ ] `resetSession()` dipanggil setiap kali pelanggan berganti.

## Stabilitas

- [ ] Hanya satu instance SDK per proses.
- [ ] `close()` dipanggil saat SDK tidak dipakai lagi.
- [ ] Setiap `PaymentListenerRegistration` dan `FlowSubscription` dibatalkan saat layar ditutup.
- [ ] WebView pembayaran dibuat sekali per sesi (dalam `remember` di Compose).
