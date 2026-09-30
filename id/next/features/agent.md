# Agent kasir

Agent kasir menerima perintah dalam bahasa sehari-hari dan menjalankan transaksi sendiri:
mencari produk, mengelola keranjang, memproses pembayaran, dan menyusun struk.

```
Kasir : cari iPhone 15, tambah 1, berapa totalnya?
Agent : iPhone 15 128GB sudah masuk keranjang. Totalnya Rp 13.999.000. Lanjut bayar?
Kasir : bayar
Agent : Silakan tempelkan kartu ke terminal...
Agent : Pembayaran berhasil (VISA). Ini struknya: ...
```

**Tersedia di:** `AIPosSDK` (artifact `aipos-sdk`) saja.

## Mengirim pesan

```kotlin
scope.launch {
    val answer: String = sdk.sendMessage("find MacBook Air then add 1")
    // answer comes back in Indonesian
}
```

`sendMessage` adalah fungsi `suspend` yang baru kembali setelah agent selesai — termasuk
setelah menunggu pelanggan menempelkan kartu. Jangan menunggunya di thread UI tanpa
indikator; amati `observeProcessing()` untuk menampilkan animasi.

## Menampilkan percakapan

```kotlin
sdk.observeMessages().collect { history ->
    history.forEach { message ->
        when (message) {
            is AgentMessage.UserMessage -> rightBubble(message.content)
            is AgentMessage.AssistantMessage -> leftBubble(message.content)
            is AgentMessage.ToolCall -> smallNote("${message.toolName}: ${message.summary}")
            is AgentMessage.ErrorMessage -> errorBubble(message.message)
        }
    }
}

sdk.observeProcessing().collect { busy -> showTypingIndicator(busy) }
```

| Tipe `AgentMessage` | Isi |
|---|---|
| `UserMessage` | Pesan yang dikirim kasir (`content`) |
| `AssistantMessage` | Jawaban agent (`content`) |
| `ToolCall` | Catatan bahwa agent menjalankan sebuah aksi (`toolName`, `summary`) — berguna untuk audit |
| `ErrorMessage` | Kegagalan saat memproses pesan (`message`) |

Semua tipe punya `timestamp` dalam milidetik epoch.

## Status pembayaran saat agent membayar

Agent baru menjawab setelah transaksi selesai. Selama menunggu, tampilkan instruksi tap
dari `observePaymentState()`:

```kotlin
sdk.observePaymentState().collect { state ->
    when (state) {
        is PaymentState.WaitingTap -> showTapDialog(state.message)
        is PaymentState.Processing -> showTapDialog(state.message)
        else -> dismissTapDialog()
    }
}
```

## Apa yang bisa dilakukan agent

| Aksi | Contoh perintah |
|---|---|
| `search_product` — cari produk berdasarkan nama atau barcode | *"ada Samsung S24?"*, *"cek 8991234500011"* |
| `manage_cart` — tambah, hapus, ubah jumlah, lihat, kosongkan | *"tambah 2"*, *"hapus Pixel-nya"*, *"berapa totalnya?"* |
| `process_payment` — pembayaran kartu contactless | *"bayar"*, *"lanjut"* |
| `generate_receipt` — susun struk teks lebar 40 karakter | *"cetak struknya"* |
| `transaction_history` — riwayat transaksi terakhir | *"transaksi hari ini apa saja?"* |

Keranjang yang diubah agent sama dengan keranjang yang diubah lewat `addToCart()` — tombol
di layar dan perintah chat bisa dipakai bergantian.

## Pengaman bawaan

- **Tidak membayar tanpa konfirmasi.** Agent diwajibkan menampilkan total lebih dulu dan
  menunggu kasir menjawab *"bayar"*, *"lanjut"*, atau *"ok"*. Aksi pembayaran menolak
  berjalan bila konfirmasi itu tidak ada, tanpa menyentuh terminal.
- **Tidak mengarang produk, harga, atau stok.** Semua angka berasal dari katalog Anda.
- **Stok divalidasi** dengan aturan yang sama seperti `addToCart()`.

## Mengawali transaksi baru

```kotlin
scope.launch { sdk.resetSession() }
```

`resetSession()` mengosongkan riwayat percakapan, saran, keranjang, dan status pembayaran
sekaligus. Panggil setiap kali pelanggan berganti.

## Batasan

- Pembayaran lewat agent hanya mendukung **kartu contactless** — tidak tersedia di iOS.
  Pembayaran online dijalankan dari tombol di layar lewat `startOnlinePayment()`.
- Membutuhkan internet dan kunci model bahasa yang valid.
- Setiap pesan memanggil model bahasa satu kali atau lebih, sehingga menimbulkan biaya
  sesuai tarif penyedia model Anda.
