# AiPosAndroid

```kotlin
object AiPosAndroid
```

Titik integrasi khusus Android untuk terminal pembayaran. Paket: `com.weekendinc.aipos`.
Artifact: `aipos-payment` (ikut di `aipos-sdk`).

## `initPlatform`

```kotlin
suspend fun initPlatform(
    application: Application,
    licenseName: String = DEFAULT_LICENSE_NAME,
): PlatformInitResult
```

Siapkan terminal pembayaran. Panggil dari `Application.onCreate`, sebelum transaksi pertama.

| Parameter | Keterangan |
|---|---|
| `application` | Instance `Application` aplikasi Anda. |
| `licenseName` | Nama file lisensi MineSec di `src/main/assets/`. |

**Mengembalikan** `PlatformInitResult(success: Boolean, message: String)`. Pada varian
simulator, selalu `success = true` dengan keterangan bahwa simulator yang dipakai.

## `attachActivity`

```kotlin
fun attachActivity(activity: ComponentActivity)
```

Daftarkan Activity yang menampilkan layar tap kartu. **Wajib** dipanggil di `onCreate`,
sebelum Activity mencapai status `STARTED`.

## Konstanta

| Konstanta | Nilai | Keterangan |
|---|---|---|
| `DEFAULT_LICENSE_NAME` | `"public-test.license"` | Lisensi lingkungan uji, sepasang dengan `MerchantInfo.DEFAULT_PROFILE_ID`. |

## Yang ikut ter-merge ke manifest

```xml
<uses-permission android:name="android.permission.NFC" />

<activity
    android:name="com.weekendinc.aipos.payment.AIPosHeadlessActivity"
    android:exported="false"
    android:launchMode="singleTask"
    android:theme="@android:style/Theme.Translucent.NoTitleBar" />
```

Jangan mengubah `launchMode` lewat `tools:replace` — MineSec menolak transaksi bila Activity
ini tidak `singleTask`.
