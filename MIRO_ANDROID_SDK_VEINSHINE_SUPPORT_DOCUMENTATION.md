# Miro Android SDK — VeinShine Scanner Support

This documentation outlines how to capture palms with an external VeinShine USB palm scanner instead of the phone camera. It supplements the [Miro Android SDK Documentation](https://github.com/mirobiometrics/Miro-Documentation/blob/main/MIRO_ANDROID_SDK_DOCUMENTATION.md), which covers enrollment, recognition, and deletion.

## Assumptions

* You have already integrated the Miro Android SDK as described in the [Android SDK Documentation](https://github.com/mirobiometrics/Miro-Documentation/blob/main/MIRO_ANDROID_SDK_DOCUMENTATION.md).
* You have a VeinShine palm scanner and a USB cable to connect it to your device.
* You have downloaded the Veinshine adapter ZIP from the Miro Admin Dashboard.

## Requirements

| Requirement | Value |
|-------------|-------|
| Device | Must support USB host mode (USB OTG) |
| ABI | `arm64-v8a` or `armeabi-v7a` |
| Permission | USB device access, requested by the SDK at capture time |
| Hardware | VeinShine palm scanner (USB vendor ID `0x3251`) |

The scanner does not use the phone camera, so the `CAMERA` permission is not required for scanner capture. You still need it if your app also supports phone-camera capture.

## Installation

Add the adapter alongside the core SDK:

```kotlin
dependencies {
    implementation("com.miro:palm-sdk:1.0.X")
    implementation("com.miro:veinshine-adapter:1.0.X")
}
```

The adapter bundles native libraries that must be extracted to disk at install time. The vendor SDK reads files from its own native-library directory and cannot load them from a compressed APK. Add this to your app's `build.gradle.kts`:

```kotlin
android {
    packaging {
        jniLibs {
            useLegacyPackaging = true
        }
    }
}
```

Omitting this causes the scanner to fail at native library load.

The adapter declares `android.hardware.usb.host` as `required="false"`, so linking it does not restrict which devices can install your app. If your app cannot function without the scanner, declare the feature yourself as required:

```xml
<uses-feature android:name="android.hardware.usb.host" android:required="true" />
```

## Enabling the Scanner

Call `install()` once, before invoking any capture method. Configuration order does not matter, but both must happen before the first capture:

```kotlin
MiroSDK.configure(MiroSDK.Credentials(
    instanceId = "your-instance-id",
    secret = "your-secret-key"
))

VeinShine.install()
```

From that point on, `enroll`, `recognize`, `verify`, and `delete` capture from the scanner. Revert to the phone camera at any time:

```kotlin
VeinShine.uninstall()
```

## Troubleshooting

| Symptom | Cause |
|---------|-------|
| Capture screen stays black, no error shown | Scanner not detected, USB permission denied, or native load failed. Diagnostics are logged under the `MiroSDK.VeinShine` tag |
| Native library load failure | `useLegacyPackaging = true` missing from the app's packaging config |
| Nothing happens on an emulator | Expected - the vendor libraries are ARM-only |
| Capture never triggers | The palm must hold alignment inside the guidance target. Reposition the hand |

## APK Size

The adapter adds no third-party dependencies, but bundles the vendor's native libraries for both supported ABIs, adding roughly 46 MB to the APK. Restrict `abiFilters` to `arm64-v8a` if you do not need 32-bit device support:

```kotlin
android {
    defaultConfig {
        ndk {
            abiFilters += listOf("arm64-v8a")
        }
    }
}
```
