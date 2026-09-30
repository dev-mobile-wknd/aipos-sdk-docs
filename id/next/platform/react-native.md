# Integrasi React Native

AI POS SDK adalah pustaka Kotlin, jadi aplikasi React Native mengaksesnya lewat
**Native Module**. Panduan ini mencakup sisi Android.

## 1. Tambahkan dependensi

```kotlin title="android/app/build.gradle.kts"
dependencies {
    implementation("com.weekendinc.aipos:aipos-sdk:0.1.1")
}
```

Pastikan `minSdk` minimal 26 dan `compileSdk` 36 — lihat [Instalasi](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/installation).
Bila memakai pembayaran kartu, daftarkan juga
[repository MineSec](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/installation#repository-minesec-pembayaran-kartu).

## 2. Inisialisasi di `MainApplication` dan `MainActivity`

```kotlin title="MainApplication.kt"
override fun onCreate() {
    super.onCreate()
    CoroutineScope(SupervisorJob() + Dispatchers.Main).launch {
        AiPosAndroid.initPlatform(this@MainApplication)
    }
    // ...other React Native initialization
}
```

```kotlin title="MainActivity.kt"
class MainActivity : ReactActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        AiPosAndroid.attachActivity(this)
    }
}
```

:::warning[`attachActivity` harus dari Activity]
Native Module hanya menerima `ReactApplicationContext`, yang bukan `ComponentActivity`.
Karena itu `attachActivity` dipanggil dari `MainActivity`, bukan dari dalam module.
:::

## 3. Buat Native Module

```kotlin title="AiPosModule.kt"
class AiPosModule(
    private val reactContext: ReactApplicationContext,
) : ReactContextBaseJavaModule(reactContext) {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    private val catalog = MutableProductCatalog()
    private var sdk: AIPosSDK? = null

    override fun getName() = "AiPos"

    @ReactMethod
    fun initialize(config: ReadableMap, promise: Promise) {
        if (sdk != null) return promise.resolve(true)

        val merchant = MerchantInfo(
            id = config.getString("merchantId").orEmpty(),
            name = config.getString("merchantName").orEmpty(),
            address = config.getString("address").orEmpty(),
            terminalId = config.getString("terminalId").orEmpty(),
            mid = config.getString("mid").orEmpty(),
            profileId = config.getString("profileId") ?: MerchantInfo.DEFAULT_PROFILE_ID,
        )

        val instance = runCatching {
            AIPosSDK.Builder()
                .llmProxyBaseUrl(config.getString("llmProxyBaseUrl").orEmpty())
                .merchantInfo(merchant)
                .productCatalog(catalog)
                .android(reactContext)
                .build()
        }.getOrElse { return promise.reject("AIPOS_CONFIG", it) }

        sdk = instance
        scope.launch {
            instance.observePaymentState().collect { state -> emit("AiPosPaymentState", state.toMap()) }
        }
        scope.launch {
            instance.observeCart().collect { cart -> emit("AiPosCart", cart.toMap()) }
        }
        promise.resolve(true)
    }

    @ReactMethod
    fun setProducts(products: ReadableArray) {
        catalog.setProducts((0 until products.size()).map { i -> products.getMap(i)!!.toProduct() })
    }

    @ReactMethod
    fun sendMessage(message: String, promise: Promise) {
        val instance = sdk ?: return promise.reject("AIPOS_NOT_READY", "Call initialize() first")
        scope.launch {
            runCatching { instance.sendMessage(message) }
                .onSuccess(promise::resolve)
                .onFailure { promise.reject("AIPOS_ERROR", it) }
        }
    }

    @ReactMethod
    fun pay() {
        val instance = sdk ?: return
        scope.launch { instance.processPayment().collect() }
    }

    @ReactMethod
    fun addListener(eventName: String) = Unit   // required for NativeEventEmitter

    @ReactMethod
    fun removeListeners(count: Int) = Unit

    override fun invalidate() {
        scope.cancel()
        super.invalidate()
    }

    private fun emit(event: String, payload: WritableMap) {
        reactContext.getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
            .emit(event, payload)
    }
}
```

Fungsi pemetaan `toMap()` dan `toProduct()` ditulis sesuai bentuk data aplikasi Anda.
Contoh untuk status pembayaran:

```kotlin
private fun PaymentState.toMap(): WritableMap = Arguments.createMap().apply {
    putString("type", this@toMap::class.simpleName)
    when (val s = this@toMap) {
        is PaymentState.WaitingTap -> putString("message", s.message)
        is PaymentState.Processing -> putString("message", s.message)
        is PaymentState.Success -> {
            putString("transactionId", s.transactionId)
            putDouble("amountRupiah", s.amount.whole.toDouble())
        }
        is PaymentState.Failed -> putString("message", s.errorMessage)
        PaymentState.Idle -> Unit
    }
}

private fun ReadableMap.toProduct() = Product(
    id = ProductId(getString("id")!!),
    name = getString("name").orEmpty(),
    price = Money.fromRupiah(getDouble("priceRupiah").toLong()),
    barcode = getString("barcode").orEmpty(),
    category = getString("category").orEmpty(),
    stock = getInt("stock"),
)
```

Daftarkan module lewat `ReactPackage` seperti Native Module pada umumnya.

## 4. Pakai dari JavaScript

```javascript
import { NativeModules, NativeEventEmitter } from 'react-native';

const { AiPos } = NativeModules;
const events = new NativeEventEmitter(AiPos);

await AiPos.initialize({
  merchantId: 'merchant_001',
  merchantName: 'Jaya Gadget Store',
  address: 'Jl. Sudirman No. 1',
  terminalId: 'TID001',
  mid: 'MID123456',
  llmProxyBaseUrl: 'https://api.tokoanda.com/llm',
});

AiPos.setProducts([
  { id: 'GDG-001', name: 'iPhone 15 128GB', priceRupiah: 13999000,
    barcode: '8991234500011', category: 'Smartphone', stock: 12 },
]);

const cartSub = events.addListener('AiPosCart', (cart) => setCart(cart));
const paySub = events.addListener('AiPosPaymentState', (state) => {
  if (state.type === 'WaitingTap') showTapPrompt(state.message);
});

const reply = await AiPos.sendMessage('find iPhone 15, add 1');

// when the screen is closed
cartSub.remove();
paySub.remove();
```

## Catatan

- Hanya satu instance SDK per proses — itulah alasan `initialize` di atas mengabaikan
  panggilan kedua.
- Untuk pembayaran online, `WebView` milik `react-native-webview` tidak mengekspos
  `evaluateJavascript` yang dibutuhkan *page probe*. Tampilkan halaman pembayaran di
  Activity atau `ViewManager` native yang memakai `AiposPaymentWebView` — lihat
  [Pembayaran online](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/features/online-payment).
