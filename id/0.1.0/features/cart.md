# Keranjang

Fungsi keranjang tersedia di `AIPosSDK` dan `PaymentClient`. Isinya disimpan di
penyimpanan lokal perangkat, jadi tetap ada meskipun aplikasi ditutup.

:::warning[`OnlinePaymentClient` belum punya fungsi keranjang]
Pada versi 0.1.0, `OnlinePaymentClient` menagih isi keranjang tetapi tidak menyediakan
`addToCart` dan kawan-kawannya, sehingga `startSession()` akan gagal dengan
*"Keranjang masih kosong"*. Untuk pembayaran online, pakai `AIPosSDK` dari `aipos-sdk`.
:::

## Mengamati isi keranjang

```kotlin
sdk.observeCart().collect { cart ->
    showTotal(cart.total.format())            // "Rp 23.498.000"
    showItemCount(cart.itemCount)             // total quantity, not the number of rows
    cart.items.forEach { item ->
        println("${item.product.name} x${item.quantity} = ${item.subtotal.format()}")
    }
}
```

## Mengubah isi keranjang

Semua operasi adalah fungsi `suspend` yang mengembalikan `PosResult<Cart>`:

```kotlin
scope.launch {
    when (val result = sdk.addToCart(product.id, quantity = 1)) {
        is PosResult.Success -> Unit                        // observeCart() updates too
        is PosResult.Failure -> showMessage(result.message)  // safe to show to the cashier
    }
}
```

| Operasi | Fungsi | Gagal bila |
|---|---|---|
| Tambah | `addToCart(productId, quantity = 1)` | Jumlah ≤ 0, produk tidak ada di katalog, stok habis, atau total melebihi stok |
| Ubah jumlah | `updateCartQuantity(productId, quantity)` | Produk tidak ada di keranjang, atau jumlah melebihi stok. Jumlah ≤ 0 menghapus baris. |
| Hapus | `removeFromCart(productId)` | Produk tidak ada di keranjang |
| Kosongkan | `clearCart()` | — |

Contoh pesan kegagalan yang bisa langsung ditampilkan:

- `iPhone 15 128GB sedang habis`
- `Stok Google Pixel 8 tidak cukup. Tersedia 5, diminta 6`
- `Produk tidak ada di keranjang`

:::tip[Cara ringkas menangani hasil]
```kotlin
sdk.addToCart(product.id)
    .onSuccess { cart -> haptic() }
    .onFailure { failure -> snackbar(failure.message) }
```
:::

## Kapan keranjang dikosongkan otomatis

| Kejadian | Keranjang |
|---|---|
| Pembayaran kartu `PaymentState.Success` | Dikosongkan, setelah transaksi tersimpan ke riwayat |
| Pembayaran online `WebPaymentStatus.Success` | Dikosongkan (sekali per pesanan) |
| Pembayaran gagal atau dibatalkan | **Tidak** dikosongkan — pelanggan bisa mencoba lagi |
| `resetSession()` di `AIPosSDK` | Dikosongkan, beserta percakapan dan saran |

## Model `Cart`

| Properti | Tipe | Keterangan |
|---|---|---|
| `items` | `List<CartItem>` | Baris belanja, urutan dipertahankan |
| `subtotal` | `Money` | Jumlah semua baris sebelum potongan |
| `discount` | `Money` | Potongan atas keseluruhan keranjang |
| `total` | `Money` | `subtotal − discount`, tidak pernah negatif |
| `itemCount` | `Int` | Total kuantitas semua produk |
| `isEmpty` | `Boolean` | `true` bila tidak ada barang |

`CartItem` berisi `product`, `quantity`, dan `subtotal` (harga × jumlah).

`Cart` bersifat *immutable*: setiap perubahan menghasilkan objek baru, jadi aman dipakai
langsung sebagai state UI.
