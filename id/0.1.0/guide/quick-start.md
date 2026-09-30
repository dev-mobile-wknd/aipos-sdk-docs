# Mulai cepat

Panduan ini membawa Anda dari proyek Android kosong sampai transaksi pertama berhasil —
tanpa terminal, tanpa lisensi, dan tanpa backend. Pembayaran kartu memakai simulator bawaan
yang selalu menyetujui transaksi.

**Yang Anda perlukan:** proyek Android dengan Jetpack Compose, dan
[SDK sudah terpasang](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/installation) (`aipos-sdk`). OpenAI API key opsional — hanya
untuk langkah 6.

## 1. Siapkan data produk

SDK tidak membawa katalog. Buat daftar produk Anda sendiri:

```kotlin
import com.weekendinc.aipos.domain.entity.Product
import com.weekendinc.aipos.domain.valueobject.Money
import com.weekendinc.aipos.domain.valueobject.ProductId

val storeProducts = listOf(
    Product(
        id = ProductId("GDG-001"),
        name = "iPhone 15 128GB",
        price = Money.fromRupiah(13_999_000),
        barcode = "8991234500011",
        category = "Smartphone",
        stock = 12,
        description = "48MP camera, USB-C",
    ),
    Product(
        id = ProductId("GDG-002"),
        name = "Google Pixel 8",
        price = Money.fromRupiah(9_499_000),
        barcode = "8991234500028",
        category = "Smartphone",
        stock = 5,
        description = "Best low-light photos in its class",
    ),
)
```

:::warning[Harga dalam Rupiah penuh]
`Money.fromRupiah(13_999_000)` berarti Rp 13.999.000. Jangan memakai konstruktor
`Money(...)` untuk harga — konstruktor itu menerima nilai dalam **sen**.
:::

## 2. Inisialisasi terminal di `Application`

```kotlin
class PosApplication : Application() {
    private val appScope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    override fun onCreate() {
        super.onCreate()
        appScope.launch {
            val result = AiPosAndroid.initPlatform(this@PosApplication)
            Log.d("POS", result.message)
        }
    }
}
```

Jangan lupa mendaftarkan kelas ini di manifest: `<application android:name=".PosApplication" ...>`.

## 3. Daftarkan Activity

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        AiPosAndroid.attachActivity(this)   // REQUIRED, and must be called in onCreate
        setContent { PosScreen() }
    }
}
```

## 4. Rakit SDK

Buat satu instance SDK per proses aplikasi. `ViewModel` adalah tempat yang wajar untuk
aplikasi satu layar:

```kotlin
class PosViewModel(application: Application) : AndroidViewModel(application) {

    private val merchant = MerchantInfo(
        id = "merchant_001",
        name = "Jaya Gadget Store",
        address = "Jl. Sudirman No. 1, Jakarta",
        terminalId = "TID001",
        mid = "MID123456",
    )

    val sdk: AIPosSDK = AIPosSDK.Builder()
        .llmApiKey(BuildConfig.LLM_API_KEY)
        .merchantInfo(merchant)
        .productCatalog(MutableProductCatalog(storeProducts))
        .android(application)
        .build()

    val catalog = sdk.observeCatalog()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

    val cart = sdk.observeCart()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), Cart())

    val paymentStatus = sdk.observePaymentState()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), PaymentState.Idle)

    fun addProduct(product: Product) = viewModelScope.launch {
        sdk.addToCart(product.id)
            .onFailure { failure -> Log.w("POS", failure.message) }
    }

    fun pay() = viewModelScope.launch {
        sdk.processPayment().collect()   // status also flows into paymentStatus
    }

    override fun onCleared() {
        // close() is a suspend function; run it in a scope that won't be cancelled.
        CoroutineScope(Dispatchers.Default).launch { sdk.close() }
    }
}
```

:::tip[Belum punya OpenAI API key?]
`build()` menolak berjalan tanpa `llmApiKey` atau `llmProxyBaseUrl`. Untuk mencoba
keranjang dan pembayaran saja, isi `llmApiKey("dummy")` — fitur non-AI tetap berjalan, dan
panggilan AI baru gagal saat benar-benar dipakai. Atau pakai
[`PaymentClient`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/reference/payment-client), yang sama sekali tidak membutuhkan kunci.
:::

## 5. Gambar layarnya

```kotlin
@Composable
fun PosScreen(vm: PosViewModel = viewModel()) {
    val catalog by vm.catalog.collectAsStateWithLifecycle()
    val cart by vm.cart.collectAsStateWithLifecycle()
    val status by vm.paymentStatus.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        catalog.forEach { product ->
            ListItem(
                headlineContent = { Text(product.name) },
                supportingContent = { Text(product.price.format()) },
                trailingContent = {
                    Button(onClick = { vm.addProduct(product) }) { Text("Add") }
                },
            )
        }

        Text("Total: ${cart.total.format()}  (${cart.itemCount} items)")

        Button(onClick = { vm.pay() }, enabled = !cart.isEmpty) {
            Text("Pay")
        }

        when (val s = status) {
            is PaymentState.WaitingTap -> Text(s.message)
            is PaymentState.Processing -> Text(s.message)
            is PaymentState.Success -> Text("Paid: ${s.amount.format()}")
            is PaymentState.Failed -> Text("Failed: ${s.errorMessage}")
            PaymentState.Idle -> Unit
        }
    }
}
```

Jalankan aplikasinya, tambahkan produk, lalu tekan **Pay**. Dalam simulator, Anda akan
melihat "[SIMULASI] Tempelkan kartu ke terminal", disusul "Memproses…", lalu status lunas.
Keranjang dikosongkan otomatis dan transaksinya tersimpan di riwayat.

## 6. Coba fitur AI

Dengan `LLM_API_KEY` yang valid, tambahkan dua fungsi ini ke ViewModel:

```kotlin
val suggestions = sdk.observeSuggestions()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

fun listen(text: String) = sdk.pushTranscript(text)

fun chat(message: String) = viewModelScope.launch {
    val answer = sdk.sendMessage(message)
    Log.d("POS", answer)
}
```

- Panggil `listen("saya cari hp buat foto-foto, budget 15 juta")` — sekitar satu setengah
  detik kemudian `suggestions` terisi produk yang cocok.
- Panggil `chat("tambahkan Pixel 8 satu")` — agent mencari produknya dan menambahkannya ke
  keranjang, yang langsung terlihat di layar.

## Selanjutnya

- [Konfigurasi](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/configuration) — kunci API, identitas merchant, pembayaran online.
- [Android](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/platform/android) — terminal sungguhan, lisensi, dan praktik terbaik.
- [Siap produksi](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guides/production-checklist) — sebelum aplikasi sampai ke kasir.
