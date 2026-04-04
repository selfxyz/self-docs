# Native Modules Setup

The Self Mobile SDK requires minimal native module configuration for Android and iOS to enable camera access and NFC scanning.

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

## iOS Setup

### 1. Enable NFC Capability

In Xcode, enable the NFC capability for your app:
1. Open your project in Xcode
2. Select your app target
3. Go to "Signing & Capabilities"
4. Click "+ Capability" and add "Near Field Communication Tag Reading"

**Important**: Set your build scheme to **Release** as Debug mode is not currently supported.

### 2. Info.plist Permissions

Add these usage descriptions to your `ios/YourApp/Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>Needed to scan the passport MRZ.</string>
<key>NSFaceIDUsageDescription</key>
<string>Needed to secure the secret</string>
<key>NFCReaderUsageDescription</key>
<string>Needed to read passport NFC chip for identity verification</string>
```

### 3. Podfile Configuration

Add the following to your `ios/Podfile` in the `post_install` block:

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |target|
    if target.name == 'mobile-sdk-alpha'
      target.build_configurations.each do |config|
        xcframework_path = "$(PODS_ROOT)/../../mobile-sdk-alpha/ios/Frameworks/NFCPassportReader.xcframework"
        modules_path_device = "#{xcframework_path}/ios-arm64/SelfSDK.framework/Modules"
        modules_path_sim = "#{xcframework_path}/ios-arm64_x86_64-simulator/SelfSDK.framework/Modules"

        # Add module search paths
        config.build_settings['OTHER_SWIFT_FLAGS'] ||= ['$(inherited)']
        config.build_settings['OTHER_SWIFT_FLAGS'] << "-I#{modules_path_device}"
        config.build_settings['OTHER_SWIFT_FLAGS'] << "-I#{modules_path_sim}"
      end
    end
  end
end
```

## Installation

```bash
npm install @selfxyz/mobile-sdk-alpha

# For iOS, install pods
cd ios && pod install && cd ..
```