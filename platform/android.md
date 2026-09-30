# Android integration

This page covers everything Android-specific: lifecycle, payment terminal, MineSec
licensing, and best practices with Jetpack Compose.

## Required steps

Two steps are tied to the Android lifecycle and can't be skipped if you use card
payment (`aipos-payment` or `aipos-sdk`).

### 1. `initPlatform` in `Application.onCreate`

MineSec validates the license and prepares cryptographic keys before the first
transaction. Start as early as possible:

```kotlin
class PosApplication : Application() {
    private val appScope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    private val _terminalReady = MutableStateFlow<PlatformInitResult?>(null)
    val terminalReady: StateFlow<PlatformInitResult?> = _terminalReady.asStateFlow()

    override fun onCreate() {
        super.onCreate()
        appScope.launch {
            _terminalReady.value = AiPosAndroid.initPlatform(
                application = this@PosApplication,
                licenseName = "tokoanda.license",   // file in src/main/assets/
            )
        }
    }
}
```

`PlatformInitResult` has `success` and `message`. Show `message` to the cashier when
`success` is `false` — for example, when the license file isn't found.

### 2. `attachActivity` in `onCreate`

The card-tap screen is launched via `registerForActivityResult`, which can only be
registered before the Activity reaches the `STARTED` state:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        AiPosAndroid.attachActivity(this)   // before setContent, not in onResume
        setContent { PosApp() }
    }
}
```

Skipping this step makes `processPayment()` emit `PaymentState.Failed` with a
message explaining why.

## Real terminal vs. simulator

The terminal variant is decided when the `aipos-payment` artifact is **built and
published**:

| Variant | Behavior |
|---|---|
| **MineSec** | The NFC phone becomes a real tap-to-pay terminal. Requires a license and `profileId`. |
| **Simulator** | No NFC. Every transaction is approved after roughly 3 seconds. Messages start with `[SIMULATED]` and `transactionId` starts with `SIM-`. |

:::danger[Make sure which variant you have]
The same Maven coordinates can contain either variant. Before shipping to a real
cashier, run one test transaction and confirm the message does **not** start with
`[SIMULATED]`. The simulator never moves real money.
:::

### MineSec requirements

1. **Registry credentials** — see [Installation](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/installation#minesec-repository-card-payment).
2. **A license file** (`*.license`) from MineSec, placed in `app/src/main/assets/`.
   The default name the SDK looks for is `AiPosAndroid.DEFAULT_LICENSE_NAME`
   (`public-test.license`, a test license).
3. **`profileId`** from the MineSec dashboard, set in [`MerchantInfo`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/configuration#merchant-identity).
4. **A device with NFC**. Add the following declaration if your app also needs to be
   installable on devices without NFC:

```xml
<uses-feature android:name="android.hardware.nfc" android:required="false" />
```

## Where to keep the SDK instance

Create **one instance per process**. Each instance stores the cart and transaction
history in local device storage, and two instances alive at the same time will
overwrite each other.

| Your app | Recommended place |
|---|---|
| A single cashier screen | That screen's `ViewModel` |
| Several screens sharing a cart | A singleton in `Application` or your DI container (Hilt/Koin) |

The SDK uses its own Koin container, so it's safe to use alongside your app's own
Koin or Hilt.

Call `close()` once the SDK is no longer needed. On `AIPosSDK`, `close()` is a
suspend function.

## Observing state with Compose

All `observe*()` functions return a `Flow`. Turn it into a `StateFlow` in the
ViewModel so the screen has an initial value right away:

```kotlin
val cart: StateFlow<Cart> = sdk.observeCart()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), Cart())
```

```kotlin
@Composable
fun CartBar(vm: PosViewModel) {
    val cart by vm.cart.collectAsStateWithLifecycle()
    Text("${cart.itemCount} items · ${cart.total.format()}")
}
```

`collectAsStateWithLifecycle` (from `androidx.lifecycle:lifecycle-runtime-compose`)
stops observing while the app is in the background.

## Calling from Java

`suspend` functions and `Flow` aren't pleasant to call from Java. Wrap them on the
Kotlin side, or use the callback provided specifically for online payment:

```kotlin
val registration = sdk.addOnlinePaymentListener { status -> /* ... */ }
registration.cancel()   // when the screen is closed
```

## Logs

The SDK writes online-payment logs with the tag `AiPos/online-payment`:

```bash
adb logcat | grep AiPos/online-payment
```

When the SDK is assembled, one log line announces the active online payment mode
(prototype or API), the backend address, and whether the key and login are set. The
password is never printed.

## Full example

The `sample-android` sample app in the SDK repository shows a complete integration:
product grid, microphone, AI suggestion chips, cart, card payment, and online
payment in a bottom sheet.
