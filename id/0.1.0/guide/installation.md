# Instalasi

AI POS SDK dipublikasikan di Maven dengan koordinat `com.weekendinc.aipos:<artifact>:0.1.0`.

## 1. Daftarkan repository

SDK dipublikasikan di **GitHub Packages privat**, bukan Maven Central — lisensinya
proprietary, bukan open source. Minta kredensial akses ke tim AI POS SDK, lalu simpan
di **gradle home** Anda — bukan di dalam proyek:

```properties
# ~/.gradle/gradle.properties
GITHUB_PACKAGES_LOGIN=<GitHub username granted access>
GITHUB_PACKAGES_TOKEN=<personal access token, scope read:packages>
```

Kemudian daftarkan repository-nya di `settings.gradle.kts`, di samping `google()`:

**settings.gradle.kts**

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            name = "AIPosSDK"
            url = uri("https://maven.pkg.github.com/dev-mobile-wknd/aipos-sdk-android")
            credentials {
                username = providers.gradleProperty("GITHUB_PACKAGES_LOGIN").get()
                password = providers.gradleProperty("GITHUB_PACKAGES_TOKEN").get()
            }
        }
    }
}
```

**settings.gradle**

```groovy
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            name = "AIPosSDK"
            url = uri("https://maven.pkg.github.com/dev-mobile-wknd/aipos-sdk-android")
            credentials {
                username = providers.gradleProperty("GITHUB_PACKAGES_LOGIN").get()
                password = providers.gradleProperty("GITHUB_PACKAGES_TOKEN").get()
            }
        }
    }
}
```

:::danger[Jangan commit kredensial]
Menaruh `GITHUB_PACKAGES_TOKEN` di `gradle.properties` milik proyek membuatnya mudah ikut
ter-commit ke Git. Simpan di `~/.gradle/gradle.properties`, atau di secret CI Anda sebagai
variabel lingkungan `ORG_GRADLE_PROJECT_GITHUB_PACKAGES_TOKEN`.
:::

### Repository MineSec (pembayaran kartu)

Lewati bagian ini bila Anda **tidak** memakai `aipos-payment` maupun `aipos-sdk`.

Kedua artifact tersebut bergantung pada MineSec Headless SDK, yang dipublikasikan di
GitHub Packages **privat**. Minta kredensial registry ke tim dukungan MineSec, lalu simpan
di **gradle home** Anda — bukan di dalam proyek:

```properties
# ~/.gradle/gradle.properties
MINESEC_REGISTRY_LOGIN=<login from MineSec>
MINESEC_REGISTRY_TOKEN=<token from MineSec>
```

Kemudian daftarkan repository-nya:

**settings.gradle.kts**

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            name = "MineSec"
            url = uri("https://maven.pkg.github.com/theminesec/ms-registry-client")
            credentials {
                username = providers.gradleProperty("MINESEC_REGISTRY_LOGIN").get()
                password = providers.gradleProperty("MINESEC_REGISTRY_TOKEN").get()
            }
        }
    }
}
```

**settings.gradle**

```groovy
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            name = "MineSec"
            url = uri("https://maven.pkg.github.com/theminesec/ms-registry-client")
            credentials {
                username = providers.gradleProperty("MINESEC_REGISTRY_LOGIN").get()
                password = providers.gradleProperty("MINESEC_REGISTRY_TOKEN").get()
            }
        }
    }
}
```

:::danger[Jangan commit kredensial]
Menaruh `MINESEC_REGISTRY_TOKEN` di `gradle.properties` milik proyek membuatnya mudah ikut
ter-commit ke Git. Simpan di `~/.gradle/gradle.properties`, atau di secret CI Anda
sebagai variabel lingkungan `ORG_GRADLE_PROJECT_MINESEC_REGISTRY_TOKEN`.
:::

## 2. Tambahkan dependensi

Pilih artifact sesuai [kebutuhan Anda](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/choosing-artifacts).

**build.gradle.kts**

```kotlin
dependencies {
    // All features
    implementation("com.weekendinc.aipos:aipos-sdk:0.1.0")

    // — or only specific features —
    // implementation("com.weekendinc.aipos:aipos-advisor:0.1.0")
    // implementation("com.weekendinc.aipos:aipos-payment:0.1.0")
    // implementation("com.weekendinc.aipos:aipos-payment-online:0.1.0")
}
```

**build.gradle**

```groovy
dependencies {
    // All features
    implementation 'com.weekendinc.aipos:aipos-sdk:0.1.0'

    // — or only specific features —
    // implementation 'com.weekendinc.aipos:aipos-advisor:0.1.0'
    // implementation 'com.weekendinc.aipos:aipos-payment:0.1.0'
    // implementation 'com.weekendinc.aipos:aipos-payment-online:0.1.0'
}
```

**gradle/libs.versions.toml**

```toml
[versions]
aipos = "0.1.0"

[libraries]
aipos-sdk = { module = "com.weekendinc.aipos:aipos-sdk", version.ref = "aipos" }
aipos-advisor = { module = "com.weekendinc.aipos:aipos-advisor", version.ref = "aipos" }
aipos-payment = { module = "com.weekendinc.aipos:aipos-payment", version.ref = "aipos" }
aipos-payment-online = { module = "com.weekendinc.aipos:aipos-payment-online", version.ref = "aipos" }

# then in build.gradle.kts:
# implementation(libs.aipos.sdk)
```

### Proyek Kotlin Multiplatform

Tambahkan dependensi di `commonMain`. Gradle memilih varian Android atau iOS secara
otomatis.

```kotlin title="shared/build.gradle.kts"
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation("com.weekendinc.aipos:aipos-sdk:0.1.0")
        }
    }
}
```

Untuk mengekspos tipe SDK ke Swift, lihat [Integrasi iOS](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/platform/ios).

## 3. Sesuaikan konfigurasi Android

SDK membutuhkan setelan minimum berikut di modul aplikasi:

```kotlin title="app/build.gradle.kts"
android {
    compileSdk = 36

    defaultConfig {
        minSdk = 26
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

kotlin {
    jvmToolchain(17)
}
```

## 4. Periksa izin di manifest

Sebagian izin sudah ikut ter-*merge* dari manifest artifact SDK:

| Izin / komponen | Didaftarkan oleh | Perlu Anda tambahkan? |
|---|---|---|
| `android.permission.INTERNET` | `aipos-llm` (ikut di `aipos-sdk`, `aipos-advisor`) | Hanya bila memakai `aipos-payment-online` saja |
| `android.permission.NFC` | `aipos-payment` | Tidak |
| Activity `AIPosHeadlessActivity` | `aipos-payment` | Tidak |
| `android.permission.RECORD_AUDIO` | — | Ya, bila memakai mikrofon untuk penyaran produk |

```xml title="AndroidManifest.xml"
<!-- Only if using aipos-payment-online without any other artifact -->
<uses-permission android:name="android.permission.INTERNET" />
```

## 5. Verifikasi

Sinkronkan Gradle, lalu pastikan dependensinya terpecahkan:

```bash
./gradlew :app:dependencies --configuration releaseRuntimeClasspath | grep aipos
```

Anda seharusnya melihat `com.weekendinc.aipos:aipos-core:0.1.0` beserta artifact yang
Anda deklarasikan.

:::tip[Selanjutnya]
Lanjutkan ke [Mulai cepat](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/guide/quick-start) untuk menjalankan transaksi pertama.
:::
