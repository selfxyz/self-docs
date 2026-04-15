# KMP SDK

{% hint style="warning" %}
The Self KMP SDK is currently in **alpha**. To get access or try it out, please [contact the Self team](https://self.xyz/#contact).
{% endhint %}

## Overview

The Self KMP SDK lets you add Self identity verification to a Kotlin Multiplatform app while keeping the verification API in shared Kotlin code.

It's a good fit if your app already uses Kotlin Multiplatform and you want to keep verification logic in shared code.

The SDK provides:
- shared configuration with `SelfSdkConfig`
- shared request construction with `VerificationRequest`
- a common launch API and callback interface

You'll need to provide:
- secure storage setup on both platforms
- Android activity binding before launch
- iOS WebView provider and secure storage via `SelfSdkSwift` or your own implementations

The SDK currently supports Android and iOS. On iOS, you also need the `SelfSdkSwift` companion package or equivalent native provider implementations.

## Quick Start

All calling code lives in your `commonMain` source set:

```kotlin
val sdk = SelfSdk.configure(
    SelfSdkConfig(
        environment = SelfEnvironment.PROD,
    )
)

val request = VerificationRequest(
    userId = "user-uuid",
    scope = "identity",
    disclosures = listOf("ofac"),
)

sdk.launch(
    request = request,
    callback = object : SelfSdkCallback {
        override fun onSuccess() {
            println("Verification completed successfully")
        }

        override fun onFailure(error: SelfSdkError) {
            println("Error [${error.code}]: ${error.message}")
        }

        override fun onCancelled() {
            println("User dismissed verification")
        }
    }
)
```

## Platform Setup

{% tabs %}
{% tab title="Android" %}

Android setup goes in your `androidMain` source set (typically your `MainActivity`).

```kotlin
import android.os.Bundle
import androidx.activity.ComponentActivity
import xyz.self.sdk.api.SelfSdk
import xyz.self.sdk.providers.EncryptedSharedPreferencesProvider
import xyz.self.sdk.providers.SdkProviderRegistry

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 1. Register providers (demo provider — implement your own for production)
        if (SdkProviderRegistry.secureStorage == null) {
            SdkProviderRegistry.secureStorage = EncryptedSharedPreferencesProvider(this)
        }

        // 2. Bind the activity (required for launching the verification screen)
        SelfSdk.bindActivity(this)

        // Now SelfSdk.configure() and sdk.launch() work from commonMain
        setContent { App() }
    }
}
```

`bindActivity()` registers an `ActivityResultLauncher` on the given `ComponentActivity`. Without it, `launch()` fails with `MISSING_ACTIVITY`.

**Convenience overload** — combines configure + bind + launch in one call:

```kotlin
SelfSdk.launch(
    activity = this,
    config = SelfSdkConfig(environment = SelfEnvironment.PROD),
    request = VerificationRequest(userId = "user-uuid"),
    callback = myCallback,
)
```

{% endtab %}

{% tab title="iOS" %}

iOS requires a Swift companion package (`SelfSdkSwift`) that provides native provider implementations using Keychain Services, CryptoKit, and WKWebView.

```swift
import SwiftUI
import ComposeApp      // Your KMP framework (exports KMP types)
import SelfSdkSwift    // Swift companion package

// Declare that Swift classes conform to KMP protocols.
// Required because Swift needs to see both the protocol (from KMP)
// and the class (from SelfSdkSwift) to establish conformance.
extension SecureStorageProviderImpl: SecureStorageProvider {}
extension WebViewProviderImpl: WebViewProvider {}

@main
struct iOSApp: App {
    init() {
        // Register providers (demo providers — implement your own for production)
        SdkProviderRegistry.shared.secureStorage = SecureStorageProviderImpl()
        IosProviderRegistry.shared.webView = WebViewProviderImpl()
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

KMP compiles Kotlin `object` singletons to Objective-C classes — Swift accesses them via `.shared`.

Once providers are registered, call `SelfSdk.configure()` and `sdk.launch()` from your shared `commonMain` code — no iOS-specific launch code is needed. The Quick Start example above works as-is on both platforms.

`SelfSdkSwift` is a convenience, not a requirement. You can provide your own classes as long as they conform to the KMP protocols (`SecureStorageProvider`, `WebViewProvider`).

{% endtab %}
{% endtabs %}

## API Reference

### SelfSdkConfig

```kotlin
SelfSdkConfig(
    endpoint: String = "https://api.self.xyz/verifier",
    environment: SelfEnvironment = SelfEnvironment.PROD,
    version: Int = 2,
    appName: String? = null,
    appEndpoint: String? = null,
    endpointType: String? = null,  // derived from environment if not set
    chainID: Int? = null,
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `endpoint` | `"https://api.self.xyz"` | Self backend API endpoint |
| `environment` | `PROD` | `SelfEnvironment.PROD` (real documents, mainnet) or `SelfEnvironment.STG` (mock documents, testnet). See the [quickstart environment guide](quickstart.md#choose-your-environment) for details. |
| `version` | `2` | Protocol version for the verification flow |
| `appName` | `null` | Display name shown in the verification UI |
| `appEndpoint` | `null` | URL of the backend verifier that verifies the ZK proof |
| `endpointType` | derived | Derived from `environment` if not set: `"https"` for PROD, `"staging_https"` for STG |
| `chainID` | `null` | Celo chain ID — `11142220` (testnet) or `42220` (mainnet) |

### VerificationRequest

```kotlin
VerificationRequest(
    userId: String? = null,
    scope: String? = null,
    disclosures: List<String> = listOf("ofac"),
    verificationId: String? = null,
    excludedCountries: List<String> = emptyList(),
    userIdType: String? = null,
    userDefinedData: String? = null,
    selfDefinedData: String? = null,
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `userId` | `null` | User identifier for the verification |
| `scope` | `null` | Verification scope (e.g., `"identity"`) |
| `disclosures` | `listOf("ofac")` | Requested disclosures (e.g., `"full_name"`, `"dob"`, `"nationality"`, `"ofac"`) |
| `verificationId` | `null` | Pre-created verification session ID |
| `excludedCountries` | `emptyList()` | Country codes to exclude from verification |
| `userIdType` | `null` | Type of user ID provided |
| `userDefinedData` | `null` | Custom data passed through the verification flow |
| `selfDefinedData` | `null` | Self-defined metadata |

### SelfSdkCallback

```kotlin
interface SelfSdkCallback {
    fun onSuccess()
    fun onFailure(error: SelfSdkError)
    fun onCancelled()
}
```

| Method | When it's called |
|--------|-----------------|
| `onSuccess()` | Verification completed successfully |
| `onFailure(error)` | Verification failed |
| `onCancelled()` | User dismissed without completing |

### SelfSdkError

```kotlin
data class SelfSdkError(
    val code: String,
    val message: String,
)
```

| Code | Platform | Meaning |
|------|----------|---------|
| `MISSING_ACTIVITY` | Android | `bindActivity()` not called before `launch()` |
| `VERIFICATION_IN_PROGRESS` | Both | A verification flow is already running |
| `LAUNCHER_NOT_AVAILABLE` | Android | Could not initialize `ActivityResultLauncher` |
| `VERIFICATION_FAILED` | Both | The verification flow failed |
| `NO_VIEW_CONTROLLER` | iOS | Could not find a top `UIViewController` to present |

## Provider Interfaces

Providers must be registered before calling `launch()`.

### Required Providers

| Provider | Registry | Platform | Purpose |
|----------|----------|----------|---------|
| `SecureStorageProvider` | `SdkProviderRegistry` | Both | Keychain/keystore read/write |
| `WebViewProvider` | `IosProviderRegistry` | iOS only | WKWebView creation and JS evaluation |

Android does not need a `WebViewProvider` — the SDK manages the WebView internally.

### SecureStorageProvider

```kotlin
interface SecureStorageProvider {
    fun get(key: String): String?
    fun set(key: String, value: String)
    fun remove(key: String)
    fun clear()
}
```

### WebViewProvider (iOS only)

```kotlin
interface WebViewProvider {
    fun createWebView(
        onMessageReceived: (String) -> Unit,
        isDebugMode: Boolean,
        queryParams: String? = null,
    ): UIView

    fun evaluateJs(js: String)
    fun getViewController(): UIViewController
    fun isBridgeRequestAllowed(): Boolean
    fun configureRemoteLoading(remoteWebAppBaseURL: String?)
    fun configureDevServer(devServerUrl: String?)
}
```

### Demo Implementations

The SDK ships demo implementations for quick prototyping. **These are not intended for production use** — you should implement your own `SecureStorageProvider` backed by your app's secure storage strategy.

**Android** (shipped in the SDK):

| Class | Backing API |
|-------|-------------|
| `EncryptedSharedPreferencesProvider` | AndroidX EncryptedSharedPreferences (AES256-SIV / AES256-GCM) |

**iOS** (shipped in `SelfSdkSwift`):

| Class | Backing API |
|-------|-------------|
| `SecureStorageProviderImpl` | Keychain Services |
| `WebViewProviderImpl` | WKWebView with `WKScriptMessageHandler` bridge |

### Implementing Your Own Providers

For production, implement the provider interfaces directly:

{% tabs %}
{% tab title="Android" %}

```kotlin
class MySecureStorage(context: Context) : SecureStorageProvider {
    override fun get(key: String): String? { /* ... */ }
    override fun set(key: String, value: String) { /* ... */ }
    override fun remove(key: String) { /* ... */ }
    override fun clear() { /* ... */ }
}

SdkProviderRegistry.secureStorage = MySecureStorage(context)
```

{% endtab %}

{% tab title="iOS" %}

```swift
class MySecureStorage: NSObject, SecureStorageProvider {
    func get(key: String) -> String? { /* ... */ }
    func set(key: String, value: String) { /* ... */ }
    func remove(key: String) { /* ... */ }
    func clear() { /* ... */ }
}

SdkProviderRegistry.shared.secureStorage = MySecureStorage()
```

{% endtab %}
{% endtabs %}
