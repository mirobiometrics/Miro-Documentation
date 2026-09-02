# Miro Android SDK — External Scanner Support

This documentation outlines how to capture palms with an external USB palm scanner instead of the phone camera. It supplements the [Miro Android SDK Documentation](https://github.com/mirobiometrics/Miro-Documentation/blob/main/MIRO_ANDROID_SDK_DOCUMENTATION.md), which covers enrollment, recognition, verification, and deletion.

Two scanners are supported, each through its own optional adapter module:

| Adapter | Hardware | Artifact |
|---------|----------|----------|
| VeinShine | VeinShine USB palm scanner | `com.miro:veinshine-adapter` |
| Armatura | Armatura PVS-50/MAP-50 palm scanner | `com.miro:armatura-adapter` |

Install one and the rest of the flow is unchanged: `enroll`, `recognize`, `verify`, and `delete` keep the same signatures, the same guidance UI, and the same results. Everything in this document that is not marked as adapter-specific applies to both.

## Assumptions

* You have already integrated the Miro Android SDK as described in the [Android SDK Documentation](https://github.com/mirobiometrics/Miro-Documentation/blob/main/MIRO_ANDROID_SDK_DOCUMENTATION.md).
* You have a supported palm scanner and a USB cable to connect it to your device.
* You have downloaded the matching adapter ZIP from the Miro Admin Dashboard.

## Requirements

| Requirement | VeinShine | ARMATURA PVS-50 |
|-------------|-----------|-----------------|
| Device | USB host mode (USB OTG) | USB host mode (USB OTG) |
| ABI | `arm64-v8a` or `armeabi-v7a` | Any — the adapter is pure Kotlin |
| Permission | USB device access, requested by the SDK at capture time | USB device access, requested by the SDK at capture time |
| Hardware | USB vendor ID `0x3251` or `0x0327` | USB vendor ID `0x34C9`, product ID `0x3240` |
| Extra build config | `useLegacyPackaging = true` (see below) | None |
| APK impact | ~46 MB of vendor native libraries | Negligible |

Neither scanner uses the phone camera, so the `CAMERA` permission is not required for scanner capture. You still need it if your app also supports phone-camera capture.

Both adapters declare `android.hardware.usb.host` as `required="false"`, so linking one does not restrict which devices can install your app. If your app cannot function without the scanner, declare the feature yourself as required:

```xml
<uses-feature android:name="android.hardware.usb.host" android:required="true" />
```

## Installation

Add the adapter you need alongside the core SDK:

```kotlin
dependencies {
    implementation("com.miro:palm-sdk:1.0.X")

    // Pick the one matching your hardware.
    implementation("com.miro:veinshine-adapter:1.0.X")
    implementation("com.miro:armatura-adapter:1.0.X")
}
```

All artifacts share the same `1.0.<build>` version; use the version in the ZIP you downloaded.

### VeinShine: Native library packaging

The VeinShine adapter bundles native libraries that must be extracted to disk at install time. The vendor SDK reads files from its own native-library directory and cannot load them from a compressed APK. Add this to your app's `build.gradle.kts`:

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

The Armatura adapter needs none of this. THe PVS-50/MAP-50 a standard UVC device driven over `UsbDeviceConnection`, so the adapter carries no native code.

## Enabling a Scanner

Call `install()` once, before invoking any capture method. Configuration order does not matter, but both must happen before the first capture:

```kotlin
MiroSDK.configure(MiroSDK.Credentials(
    instanceId = "your-instance-id",
    secret = "your-secret-key"
))

VeinShine.install()   // or Armatura.install()
```

From that point on, `enroll`, `recognize`, `verify`, and `delete` capture from the scanner. Revert to the phone camera at any time:

```kotlin
VeinShine.uninstall()   // or Armatura.uninstall()
```

Each adapter reports its own optics to the SDK, so guidance and the upload adapt to the hardware without any configuration on the integration side. In particular the scanner's mirroring is declared by the adapter, so `CaptureConfig.imageMirrored` does not need to be set for scanner capture.

**Only one capture source is active at a time.** The two adapters share a single slot in the SDK, so installing one replaces whatever was installed before it. `uninstall()` only clears the slot if that adapter still owns it, so calling it on an adapter that has since been replaced is harmless.

## Troubleshooting

| Symptom | Cause |
|---------|-------|
| Capture screen stays black, no error shown | Scanner not detected, USB permission denied, or (VeinShine) native load failed. Diagnostics are logged under the `MiroSDK.VeinShine` / `MiroSDK.Armatura` tag |
| Native library load failure (VeinShine) | `useLegacyPackaging = true` missing from the app's packaging config |
| Nothing happens on an emulator | Expected — there is no USB host bus, and the VeinShine vendor libraries are ARM-only |
| Capture never triggers | The palm must hold alignment inside the guidance target. Reposition the hand |
| "No PVS-50 found on the USB bus" (Armatura) | The scanner is not enumerated. Check the OTG cable and that the device supplies enough power |
| Preview is upside down or NIR/visible look swapped (Armatura) | The two video functions are reversed on that unit — set `Armatura.swapVisibleAndNir = true` |

## APK Size

The Armatura adapter adds no third-party dependencies and no native code, so its footprint is negligible.

The VeinShine adapter also adds no third-party dependencies, but bundles the vendor's native libraries for both supported ABIs, adding roughly 46 MB to the APK. Restrict `abiFilters` to `arm64-v8a` if you do not need 32-bit device support:

```kotlin
android {
    defaultConfig {
        ndk {
            abiFilters += listOf("arm64-v8a")
        }
    }
}
```
