# Miro Android SDK: Scanner Support

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
| ABI | `arm64-v8a` or `armeabi-v7a` | Any, since the adapter is pure Kotlin |
| Permission | USB device access, requested by the SDK at capture time | USB device access, plus runtime `CAMERA` on Android 10+ (see [USB Permission](#usb-permission)) |
| Hardware | USB vendor ID `0x3251` or `0x0327` | USB vendor ID `0x34C9`, product ID `0x3240` |
| Extra build config | `useLegacyPackaging = true` (see below) | None |
| APK impact | ~46 MB of vendor native libraries | Negligible |

The VeinShine scanner does not use the phone camera, so the `CAMERA` permission is not required for VeinShine capture. **The ARMATURA PVS-50 is different:** it enumerates as a USB video-class (UVC) device, and from Android 10 the platform refuses USB access to a video-class device for an app that does not hold runtime `CAMERA`; it sends a denial without ever showing a dialog. Request `CAMERA` before the first Armatura capture even though the phone camera is never used.

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

## USB Permission

Android grants an app access to a USB device in one of two ways, and which one you get is decided by your manifest, not by the SDK.

By default the adapter asks for access when a capture starts, using `UsbManager.requestPermission()`. That grant lasts only as long as the current connection: Unplugging the scanner revokes it. The next capture prompts again, and because the request happens inside the capture flow, the system dialog appears on top of the capture screen.

Adding an attach filter switches this to a persistent grant. The dialog then carries a *"Use by default for this USB device"* checkbox, and once the user ticks it the app is granted access on every future attach with no dialog at all.

### What you need to add

**1. A device filter listing the scanners you support.** Create `res/xml/miro_scanner_device_filter.xml` in your app and paste the one that matches your hardware. IDs are decimal, since Android's `<usb-device>` does not accept hex.

ARMATURA only:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- ARMATURA PVS-50/MAP-50: vendor 0x34C9, product 0x3240 -->
    <usb-device vendor-id="13513" product-id="12864" />
</resources>
```

VeinShine only:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- VeinShine: vendor 0x3251 -->
    <usb-device vendor-id="12881" />
    <!-- VeinShine: vendor 0x0327 -->
    <usb-device vendor-id="807" />
</resources>
```

Both scanners in one app:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- ARMATURA PVS-50/MAP-50: vendor 0x34C9, product 0x3240 -->
    <usb-device vendor-id="13513" product-id="12864" />

    <!-- VeinShine: vendor 0x3251 -->
    <usb-device vendor-id="12881" />
    <!-- VeinShine: vendor 0x0327 -->
    <usb-device vendor-id="807" />
</resources>
```

Supporting both scanners means one file with every device in it. A `meta-data` entry takes exactly one resource, so pointing at two filters is a manifest merger error rather than an override.

The VeinShine entries match on vendor alone because that scanner ships under more than one product ID; the adapter narrows it further by product name at connect time. Listing a vendor without a product ID means your app is offered as a handler for any device from that vendor, which is why the ARMATURA entry pins both.

**2. An attach filter on the activity that hosts capture.** This has to live in your manifest, since an `<intent-filter>` attaches to a concrete component, and the SDK ships no activity of its own:

```xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleTop"
    android:exported="true">

    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>

    <intent-filter>
        <action android:name="android.hardware.usb.action.USB_DEVICE_ATTACHED" />
    </intent-filter>
    <meta-data
        android:name="android.hardware.usb.action.USB_DEVICE_ATTACHED"
        android:resource="@xml/miro_scanner_device_filter" />
</activity>
```

**3. Ignore the attach intent.** `singleTop` keeps the attach event from stacking a second copy of your activity over a capture in progress, and the activity has nothing to do with the intent itself:

```kotlin
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    if (intent.action == UsbManager.ACTION_USB_DEVICE_ATTACHED) return
    setIntent(intent)
}
```

**4. Ask for access before capture, not during it (Armatura only).** This moves the dialog somewhere you can explain it, instead of onto the capture screen where a denial reads as the capture having failed:

```kotlin
when (Armatura.requestAccess(this)) {
    Armatura.AccessState.GRANTED   -> Unit        // already permitted; go ahead
    Armatura.AccessState.REQUESTED -> Unit        // dialog is up; proceed, capture waits for it
    Armatura.AccessState.NO_DEVICE -> showMessage("Connect the palm scanner")
    Armatura.AccessState.NEEDS_CAMERA_PERMISSION ->
        requestCameraPermission()                 // then call requestAccess again
    Armatura.AccessState.UNSUPPORTED -> hideScanner()   // no USB host support
}
```

Call it once you hold runtime `CAMERA` and have a foreground activity. `Application.onCreate` is too early for the dialog, and on Android 10+ a call without `CAMERA` returns `NEEDS_CAMERA_PERMISSION` and does nothing. It performs a few synchronous binder calls, so it is not something to call per screen.

There is no VeinShine equivalent yet; for VeinShine, add the attach filter and let the adapter request access at capture time.

### If you change nothing

Everything keeps working, since none of this is required for capture to succeed. What you get instead:

| Scenario | Experience with no manifest or code changes |
|----------|--------------------------------------------|
| First capture after install | System USB dialog appears **on top of the capture screen**, with no explanation of what is being asked |
| User grants, then keeps the scanner plugged in | No further prompts for the life of the connection |
| Scanner unplugged and reconnected | Prompt returns, again over the capture screen, once per plug cycle |
| Device rebooted with scanner attached | Prompt returns |
| User denies | Capture shows *"USB permission denied — re-plug and allow"* and does not proceed |
| Armatura without runtime `CAMERA` (Android 10+) | Access is refused **silently, with no dialog**, capture fails with no visible reason |

The recurring prompt is the usual complaint from the field, and it is entirely a manifest matter: without the attach filter there is no mechanism for a grant to outlive the connection.

## Troubleshooting

| Symptom | Cause |
|---------|-------|
| Capture screen stays black, no error shown | Scanner not detected, USB permission denied, or (VeinShine) native load failed. Diagnostics are logged under the `MiroSDK.VeinShine` / `MiroSDK.Armatura` tag |
| USB permission prompt appears on every plug cycle | No attach filter in your manifest, see [USB Permission](#usb-permission) |
| Armatura capture fails with no dialog and no error (Android 10+) | Runtime `CAMERA` is not granted. The PVS-50 is a video-class device, so the platform refuses USB access silently. Request `CAMERA`, then call `Armatura.requestAccess()` |
| Native library load failure (VeinShine) | `useLegacyPackaging = true` missing from the app's packaging config |
| Nothing happens on an emulator | Expected, since there is no USB host bus, and the VeinShine vendor libraries are ARM-only |
| Capture never triggers | The palm must hold alignment inside the guidance target. Reposition the hand |
| "No PVS-50 found on the USB bus" (Armatura) | The scanner is not enumerated. Check the OTG cable and that the device supplies enough power |
| Preview is upside down or NIR/visible look swapped (Armatura) | The two video functions are reversed on that unit, set `Armatura.swapVisibleAndNir = true` |

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
