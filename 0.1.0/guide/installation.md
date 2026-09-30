# Installation

AI POS SDK is published on Maven with the coordinates `com.weekendinc.aipos:<artifact>:0.1.0`.

## 1. Register the repository

The SDK is published on **private GitHub Packages**, not Maven Central — the license is
proprietary, not open source. Ask the AI POS SDK team for access credentials, then store
them in your **gradle home** — not inside the project:

```properties
# ~/.gradle/gradle.properties
GITHUB_PACKAGES_LOGIN=<GitHub username granted access>
GITHUB_PACKAGES_TOKEN=<personal access token, scope read:packages>
```

Then register the repository in `settings.gradle.kts`, alongside `google()`:

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

:::danger[Don't commit credentials]
Putting `GITHUB_PACKAGES_TOKEN` in the project's own `gradle.properties` makes it easy
to accidentally commit to Git. Store it in `~/.gradle/gradle.properties`, or in your
CI secrets as the environment variable `ORG_GRADLE_PROJECT_GITHUB_PACKAGES_TOKEN`.
:::

### MineSec repository (card payment)

Skip this section if you use **neither** `aipos-payment` nor `aipos-sdk`.

Both artifacts depend on the MineSec Headless SDK, which is published on **private**
GitHub Packages. Ask MineSec support for registry credentials, then store them in your
**gradle home** — not inside the project:

```properties
# ~/.gradle/gradle.properties
MINESEC_REGISTRY_LOGIN=<login from MineSec>
MINESEC_REGISTRY_TOKEN=<token from MineSec>
```

Then register the repository:

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

:::danger[Don't commit credentials]
Putting `MINESEC_REGISTRY_TOKEN` in the project's own `gradle.properties` makes it easy
to accidentally commit to Git. Store it in `~/.gradle/gradle.properties`, or in your
CI secrets as the environment variable `ORG_GRADLE_PROJECT_MINESEC_REGISTRY_TOKEN`.
:::

## 2. Add the dependency

Choose the artifact that matches [your needs](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/choosing-artifacts).

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

### Kotlin Multiplatform project

Add the dependency in `commonMain`. Gradle automatically picks the Android or iOS
variant.

```kotlin title="shared/build.gradle.kts"
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation("com.weekendinc.aipos:aipos-sdk:0.1.0")
        }
    }
}
```

To expose the SDK's types to Swift, see [iOS integration](https://dev-mobile-wknd.github.io/aipos-sdk-docs/platform/ios).

## 3. Adjust the Android configuration

The SDK requires the following minimum settings in your app module:

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

## 4. Check manifest permissions

Some permissions are already merged in from the SDK artifacts' manifests:

| Permission / component | Registered by | Do you need to add it? |
|---|---|---|
| `android.permission.INTERNET` | `aipos-llm` (pulled in by `aipos-sdk`, `aipos-advisor`) | Only if using `aipos-payment-online` alone |
| `android.permission.NFC` | `aipos-payment` | No |
| `AIPosHeadlessActivity` Activity | `aipos-payment` | No |
| `android.permission.RECORD_AUDIO` | — | Yes, if using the microphone for product suggestions |

```xml title="AndroidManifest.xml"
<!-- Only if using aipos-payment-online without any other artifact -->
<uses-permission android:name="android.permission.INTERNET" />
```

## 5. Verify

Sync Gradle, then confirm the dependencies resolve:

```bash
./gradlew :app:dependencies --configuration releaseRuntimeClasspath | grep aipos
```

You should see `com.weekendinc.aipos:aipos-core:0.1.0` along with the artifacts
you declared.

:::tip[Next]
Continue to [Quick start](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/quick-start) to run your first transaction.
:::
