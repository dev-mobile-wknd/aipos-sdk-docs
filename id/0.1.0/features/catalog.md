# Katalog produk

SDK **tidak membawa data produk**. Aplikasi Anda menyerahkan katalog lewat
`ProductCatalogSource`, dan SDK hanya membacanya untuk:

- menggambar daftar produk lewat `observeCatalog()`,
- mencari produk saat agent kasir menerima perintah,
- menyusun saran produk — hanya barang yang ada di katalog yang boleh disarankan,
- memvalidasi stok saat barang ditambahkan ke keranjang.

## Kontraknya

```kotlin
fun interface ProductCatalogSource {
    fun observeProducts(): Flow<List<Product>>
}
```

Bentuknya stream. Perubahan di sisi Anda — stok berkurang, harga naik, produk baru —
langsung terlihat SDK tanpa perlu merakit ulang.

## Pilihan 1: `MutableProductCatalog`

Untuk katalog statis, atau bila Anda lebih suka mendorong data:

```kotlin
val catalog = MutableProductCatalog(initialProducts)

val sdk = AIPosSDK.Builder()
    .productCatalog(catalog)
    // ...
    .build()

// Whenever the data changes, e.g. after syncing with the server:
catalog.setProducts(latestProducts)
```

`setProducts` **mengganti seluruh isi** katalog, bukan menambah. Produk yang tidak ada di
daftar baru langsung hilang dari pencarian dan saran. Aman dipanggil dari thread mana pun.

## Pilihan 2: implementasi sendiri

Bila katalog Anda sudah berupa stream — Room, SQLDelight, atau `StateFlow` di repository —
sambungkan langsung supaya tidak ada salinan kedua yang bisa kedaluwarsa:

```kotlin
val catalog = ProductCatalogSource {
    productDao.observeAll().map { rows -> rows.map { it.toAiposProduct() } }
}

private fun ProductEntity.toAiposProduct() = Product(
    id = ProductId(sku),
    name = name,
    price = Money.fromRupiah(priceRupiah),
    barcode = barcode.orEmpty(),
    category = category,
    stock = stock,
    description = shortDescription.orEmpty(),
    imageUrl = imageUrl.orEmpty(),
)
```

:::warning[Dua syarat untuk implementasi sendiri]
1. **Flow wajib memancarkan nilai pertama segera** — daftar kosong pun boleh. SDK menunggu
   emisi pertama saat menambah barang ke keranjang; flow yang tidak pernah memancar akan
   membuat kasir menunggu tanpa akhir.
2. **Flow harus murah dikoleksi.** SDK mengoleksinya ulang setiap kali butuh katalog.
   Pakai `StateFlow`, flow database, atau cache — jangan memanggil jaringan pada setiap
   koleksi.
:::

## Model `Product`

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | `ProductId` | Identitas unik. Tidak boleh kosong. |
| `name` | `String` | Nama tampilan. Dipakai agent saat mencari produk. |
| `price` | `Money` | Harga satuan. Buat dengan `Money.fromRupiah(...)`. |
| `barcode` | `String` | Kode batang (EAN-13), untuk pencarian lewat pemindai. |
| `category` | `String` | Misalnya `"Smartphone"` atau `"Laptop"`. |
| `stock` | `Int` | Sisa stok. Nol berarti habis dan tidak bisa ditambahkan ke keranjang. |
| `description` | `String` | Ringkasan satu baris. **Sangat memengaruhi kualitas saran AI** — isi dengan keunggulan produk. |
| `imageUrl` | `String` | Alamat gambar. SDK tidak pernah memuatnya; tampilkan dengan pemuat gambar Anda. |

:::tip[Deskripsi yang baik menghasilkan saran yang baik]
Penyaran produk mencocokkan kebutuhan pelanggan dengan `name`, `category`, `price`, dan
`description`. Deskripsi seperti *"Kamera 48MP, baterai 2 hari, tahan air IP68"* jauh lebih
berguna daripada *"HP bagus"*.
:::

## Stok tetap tanggung jawab Anda

SDK memvalidasi permintaan terhadap angka `stock` yang Anda berikan, tetapi **tidak pernah
mengurangi stok**. Setelah transaksi lunas, catat penjualan dan perbarui stok di sistem Anda.
Karena katalog berupa stream, angka baru langsung dipakai SDK:

```kotlin
sdk.processPayment().collect { state ->
    if (state is PaymentState.Success) {
        val sold = sdk.observeTransactions(limit = 1).first().first().cart
        inventory.reduceStock(sold.items)   // your own app's code
    }
}
```
