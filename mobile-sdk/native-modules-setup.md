# Native Modules Setup

The Self Mobile SDK requires minimal native module configuration for Android to enable camera access and NFC scanning.

## Android Setup

### 1. MainActivity Configuration

Add the import and NFC intent handling to your `android/app/src/main/java/com/yourapp/MainActivity.kt`:

```kotlin
import com.selfxyz.selfSDK.RNSelfPassportReaderModule
import android.content.Intent
import android.util.Log

class MainActivity : ReactActivity() {
    
    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        Log.d("MAIN_ACTIVITY", "onNewIntent: " + intent.action)
        try {
            RNSelfPassportReaderModule.getInstance().receiveIntent(intent)
        } catch (e: IllegalStateException) {
            Log.w("MAIN_ACTIVITY", "RNSelfPassportReaderModule not ready; deferring NFC intent")
            setIntent(intent)
        }
    }
}
```

### 2. MainApplication Configuration

Add the SelfSDK package to your `android/app/src/main/java/com/yourapp/MainApplication.kt`:

```kotlin
import com.selfxyz.selfSDK.RNSelfPassportReaderPackage

class MainApplication : Application(), ReactApplication {
    
    override fun getPackages(): List<ReactPackage> {
        val packages = PackageList(this).packages.toMutableList()
        
        packages.add(RNSelfPassportReaderPackage())
        return packages
    }
}
```

### 3. Android Manifest Permissions

Add these permissions to `android/app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.NFC" />
<uses-permission android:name="android.permission.VIBRATE" />
```

### 4. NFC Meta Data

Add this meta-data to your MainActivity in `AndroidManifest.xml`:

```xml
<meta-data
  android:name="android.nfc.action.TECH_DISCOVERED"
  android:resource="@xml/nfc_tech_filter" />
```

Create the NFC tech filter file at `android/app/src/main/res/xml/nfc_tech_filter.xml`:

```xml
<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">
    <tech-list>
        <tech>android.nfc.tech.IsoDep</tech>
    </tech-list>
</resources>
```

### 5. Build Configuration

Add this to your `android/app/build.gradle`:

```gradle
apply from: file("../../node_modules/@selfxyz/mobile-sdk-alpha/android/mobile-sdk-alpha-bom.gradle")

android {
    buildFeatures {
        viewBinding true
    }
}
```