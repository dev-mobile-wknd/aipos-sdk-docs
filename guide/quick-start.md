# Quick start

This guide takes you from an empty Android project to a successful first transaction —
no terminal, no license, and no backend. Card payment uses the built-in simulator,
which always approves the transaction.

**What you need:** an Android project with Jetpack Compose, and the
[SDK already installed](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/installation) (`aipos-sdk`). An OpenAI API key is
optional — only needed for step 6.

## 1. Prepare product data

The SDK doesn't ship a catalog. Build your own product list:

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

:::warning[Prices are in whole Rupiah]
`Money.fromRupiah(13_999_000)` means Rp 13,999,000. Don't use the `Money(...)`
constructor for prices — that constructor takes a value in **cents**.
:::

## 2. Initialize the terminal in `Application`

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

Don't forget to register this class in the manifest: `<application android:name=".PosApplication" ...>`.

## 3. Register the Activity

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        AiPosAndroid.attachActivity(this)   // REQUIRED, and must be called in onCreate
        setContent { PosScreen() }
    }
}
```

## 4. Assemble the SDK

Create one SDK instance per app process. A `ViewModel` is a reasonable place for a
single-screen app:

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

:::tip[Don't have an OpenAI API key yet?]
`build()` refuses to run without `llmApiKey` or `llmProxyBaseUrl`. To try just the
cart and payment, pass `llmApiKey("dummy")` — non-AI features keep working, and only
AI calls fail when actually used. Or use
[`PaymentClient`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/reference/payment-client), which doesn't need a key at all.
:::

## 5. Draw the screen

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

Run the app, add a product, then tap **Pay**. In the simulator, you'll see
"[SIMULATED] Tap a card on the terminal", followed by "Processing…", then a paid
status. The cart is cleared automatically and the transaction is saved to history.

## 6. Try the AI features

With a valid `LLM_API_KEY`, add these two functions to the ViewModel:

```kotlin
val suggestions = sdk.observeSuggestions()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

fun listen(text: String) = sdk.pushTranscript(text)

fun chat(message: String) = viewModelScope.launch {
    val answer = sdk.sendMessage(message)
    Log.d("POS", answer)
}
```

- Call `listen("looking for a phone for photography, budget 15 million")` — about a
  second and a half later, `suggestions` fills with matching products.
- Call `chat("add one Pixel 8")` — the agent finds the product and adds it to the
  cart, which is immediately visible on screen.

## Next

- [Configuration](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/configuration) — API key, merchant identity, online payment.
- [Android](https://dev-mobile-wknd.github.io/aipos-sdk-docs/platform/android) — real terminals, licensing, and best practices.
- [Production checklist](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guides/production-checklist) — before your app reaches a real cashier.
