# AiPosAndroid

```kotlin
object AiPosAndroid
```

The Android-specific integration point for the payment terminal. Package:
`com.weekendinc.aipos`. Artifact: `aipos-payment` (pulled in by `aipos-sdk`).

## `initPlatform`

```kotlin
suspend fun initPlatform(
    application: Application,
    licenseName: String = DEFAULT_LICENSE_NAME,
): PlatformInitResult
```

Prepares the payment terminal. Call it from `Application.onCreate`, before the
first transaction.

| Parameter | Description |
|---|---|
| `application` | Your app's `Application` instance. |
| `licenseName` | The MineSec license file's name in `src/main/assets/`. |

**Returns** `PlatformInitResult(success: Boolean, message: String)`. On the
simulator variant, always `success = true`, noting that the simulator is in use.

## `attachActivity`

```kotlin
fun attachActivity(activity: ComponentActivity)
```

Registers the Activity that shows the card-tap screen. **Required**, must be
called in `onCreate`, before the Activity reaches the `STARTED` state.

## Constants

| Constant | Value | Description |
|---|---|---|
| `DEFAULT_LICENSE_NAME` | `"public-test.license"` | A test-environment license, paired with `MerchantInfo.DEFAULT_PROFILE_ID`. |

## What gets merged into the manifest

```xml
<uses-permission android:name="android.permission.NFC" />

<activity
    android:name="com.weekendinc.aipos.payment.AIPosHeadlessActivity"
    android:exported="false"
    android:launchMode="singleTask"
    android:theme="@android:style/Theme.Translucent.NoTitleBar" />
```

Don't change `launchMode` via `tools:replace` — MineSec rejects transactions if
this Activity isn't `singleTask`.
