# Miro iOS SDK Documentation

This documentation outlines how to integrate the [Miro](https://mirobiometrics.com/) iOS SDK for palm biometric enrollment, recognition, verification, and deletion. The SDK handles camera capture, palm detection, image encryption, and API communication.

## Assumptions

* This documentation assumes you have set up a Miro Identity Instance on the [Miro Admin Dashboard](https://dashboard.mirobiometrics.com/).
* You have your Instance ID and Secret from the dashboard.
* You have downloaded the SDK ZIP from the Miro Admin Dashboard.

## Requirements

| Requirement | Value |
|-------------|-------|
| Min iOS | 16.0 |
| Language | Swift 5.9+ |
| Xcode | 16 or later |
| UI framework | UIKit or SwiftUI (both supported) |
| Permission | `NSCameraUsageDescription` in your `Info.plist` |
| Device | Physical device required — the iOS Simulator has no camera |

The SDK requests camera permission itself and returns a failure result if it is denied, so you do not need to request it up front. The `Info.plist` key is still mandatory: iOS terminates the app on first camera access without it.

```xml
<key>NSCameraUsageDescription</key>
<string>The camera is used to capture your palm for biometric enrollment and recognition.</string>
```

## Installation

The SDK ships as an XCFramework inside the ZIP you downloaded from the Miro Admin Dashboard.

1. Unzip the archive and drag `MiroPalmSDK.xcframework` into your Xcode project.
2. Select your app target, open **General → Frameworks, Libraries, and Embedded Content**, and set `MiroPalmSDK.xcframework` to **Embed & Sign**.

The XCFramework is self-contained: ONNX Runtime is statically linked and the palm detection model travels in the framework's own resource bundle. There are no additional dependencies to add and nothing to resolve over the network.

It contains slices for both physical devices and the simulator, so it links against either destination — though capture itself requires a physical device, as noted above.

To take a new SDK version, replace the `.xcframework` directory with the one from the newer ZIP.

## Configuration

Configure the SDK with your credentials before making any API calls. This should be done once, typically when the user provides or your app loads the stored credentials:

```swift
import MiroPalmSDK

MiroSDK.shared.configure(
    credentials: MiroSDK.Credentials(
        instanceId: "your-instance-id",
        secret: "your-secret-key"
    )
)
```

### Capture options

Pass a `CaptureConfig` to change which lens is used:

```swift
MiroSDK.shared.configure(
    credentials: MiroSDK.Credentials(instanceId: "...", secret: "..."),
    captureConfig: MiroSDK.CaptureConfig(cameraPosition: .front)
)
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `cameraPosition` | `CameraPosition` | `.back` | `.back` or `.front`. The back lens is strongly preferred — it has a torch, and palm ridge contrast depends on it. |
| `imageMirrored` | `Bool` | `false` | Forces the mirrored flag on regardless of lens. The front lens already reports itself as mirrored, so this is only needed for an unusual optical path that also mirrors. |

## API Methods

All SDK methods are `async` and main-actor isolated, so they must be called from a `Task` or another `async` context on the main actor. Each method presents the camera capture UI, guides the user through palm positioning, and then calls the Miro API.

Each method takes a `presenting` view controller, which the capture screen is presented from.

### Enroll

Creates a biometric profile from a palm image. Captures the first palm, then offers a second — enrolling both hands materially improves later recognition, so the prompt is shown either way.

```swift
Task {
    let result = await MiroSDK.shared.enroll(
        presenting: self,
        customerId: "user-123",          // optional
        customerData: "encrypted-data"   // optional
    )
    handleResult(result)
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `presenting` | `UIViewController` | Yes | The controller to present the capture screen from |
| `customerId` | `String?` | No | Unique customer identifier to associate with the profile |
| `customerData` | `String?` | No | Encrypted customer data to store with the profile |
| `requireTwoPalms` | `Bool` | No | When `true`, the second palm cannot be skipped. Defaults to `false` |

### Recognize

Matches a palm against enrolled profiles in your Identity Instance.

```swift
Task {
    let result = await MiroSDK.shared.recognize(presenting: self)
    handleResult(result)
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `presenting` | `UIViewController` | Yes | The controller to present the capture screen from |
| `confidenceTiers` | `Bool` | No | When `true`, the backend grades the match into `HIGH`/`MEDIUM`/`LOW` and returns it in `confidenceTier`. Defaults to `false` |

### Verify

Checks a palm against one claimed identity, rather than searching every profile.

```swift
Task {
    let result = await MiroSDK.shared.verify(
        presenting: self,
        type: .customerId,
        id: "user-123"
    )
    handleResult(result)
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `presenting` | `UIViewController` | Yes | The controller to present the capture screen from |
| `type` | `VerificationType` | Yes | `.profileId` or `.customerId` — which identifier `id` refers to |
| `id` | `String` | Yes | The identifier to verify the palm against |

A completed check that does **not** match returns `.failure` with error `VERIFICATION_FAILED`. That is a normal, expected answer rather than a fault — branch on the error code, not on the result type alone.

### Delete

Removes a profile by palm match from your Identity Instance.

```swift
Task {
    let result = await MiroSDK.shared.delete(presenting: self)
    handleResult(result)
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `presenting` | `UIViewController` | Yes | The controller to present the capture screen from |

## SwiftUI

SwiftUI views have no view controller to present from, so the SDK provides a `MiroPalm` wrapper that resolves one from the active scene. Configuration, result types, and error codes are all shared with the UIKit API — there is no second surface to keep in sync.

```swift
import MiroPalmSDK
import SwiftUI

struct ContentView: View {
    @State private var result: MiroSDK.SdkResult?

    var body: some View {
        VStack(spacing: 16) {
            Button("Enroll") {
                Task { result = await MiroPalm.enroll(customerId: "user-123") }
            }
            Button("Recognize") {
                Task { result = await MiroPalm.recognize() }
            }
            Button("Verify") {
                Task { result = await MiroPalm.verify(type: .customerId, id: "user-123") }
            }
            Button("Delete") {
                Task { result = await MiroPalm.delete() }
            }

            if let result {
                Text(result.toJSON())
                    .font(.system(.caption, design: .monospaced))
            }
        }
    }
}
```

`MiroPalm` mirrors every method on `MiroSDK.shared` with the `presenting` parameter removed. Use `MiroSDK.shared` directly when you already hold a view controller.

## Handling Results

All methods return a `MiroSDK.SdkResult`:

```swift
func handleResult(_ result: MiroSDK.SdkResult) {
    switch result.type {
    case .success:
        // result.profileId      - matched/created profile ID
        // result.customerId     - associated customer ID (if set)
        // result.requestId      - unique request identifier
        // result.confidenceTier - match grade, if confidenceTiers was requested
        break
    case .failure:
        // result.error  - error code (e.g. "IMAGE_QUALITY", "MISSING_CREDENTIALS")
        // result.detail - human-readable description
        break
    case .cancelled:
        // User dismissed the capture UI
        break
    }
}
```

**Result Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `type` | `ResultType` | `.success`, `.failure`, or `.cancelled` |
| `profileId` | `String?` | The biometric profile ID (on success) |
| `customerId` | `String?` | Customer ID associated with the profile |
| `customerData` | `String?` | Customer data associated with the profile |
| `requestId` | `String?` | Unique request identifier. Quote this when raising a support issue |
| `error` | `String?` | Error code (on failure) |
| `detail` | `String?` | Error description (on failure) |
| `confidenceTier` | `ConfidenceTier?` | `.high`, `.medium`, or `.low`, when `confidenceTiers` was requested |
| `toJSON()` | `String` | Pretty-printed JSON of the result, for logging |

**Error Codes:**

| Error Code | Description |
|------------|-------------|
| `MISSING_CREDENTIALS` | `MiroSDK.shared.configure()` was not called |
| `KEY_EXCHANGE_FAILED` | Failed to fetch RSA public key from server |
| `IMAGE_QUALITY` | Captured image failed quality checks (too blurry, too dark, or too bright) |
| `CAPTURE_CANCELLED` | User dismissed the capture UI |
| `VERIFICATION_FAILED` | The verification check completed and the palm is not the claimed identity |
| `ENCRYPTION_FAILED` | The image could not be encrypted on device |
| `SDK_UPDATE_REQUIRED` | The backend no longer supports this SDK version — ship an update |
| `NO_PRESENTER` | `MiroPalm` could not find a view controller in the active scene |

## Full Example

A complete view controller demonstrating all four operations:

```swift
import MiroPalmSDK
import UIKit

class MainViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()

        MiroSDK.shared.configure(
            credentials: MiroSDK.Credentials(
                instanceId: "<instance-id>",
                secret: "<secret>"
            )
        )
    }

    @IBAction func enrollTapped() {
        Task {
            let result = await MiroSDK.shared.enroll(
                presenting: self,
                customerId: "user-123"
            )
            handleResult(result)
        }
    }

    @IBAction func recognizeTapped() {
        Task {
            handleResult(await MiroSDK.shared.recognize(presenting: self))
        }
    }

    @IBAction func verifyTapped() {
        Task {
            let result = await MiroSDK.shared.verify(
                presenting: self,
                type: .customerId,
                id: "user-123"
            )
            handleResult(result)
        }
    }

    @IBAction func deleteTapped() {
        Task {
            handleResult(await MiroSDK.shared.delete(presenting: self))
        }
    }

    private func handleResult(_ result: MiroSDK.SdkResult) {
        switch result.type {
        case .success:
            print("Miro: profile \(result.profileId ?? "-")")
        case .failure:
            print("Miro: error \(result.error ?? "-") - \(result.detail ?? "-")")
        case .cancelled:
            print("Miro: cancelled")
        }
    }
}
```

## Capture Behaviour

The capture screen is presented full-screen and locked to portrait for its lifetime. It handles the whole capture loop for you:

* A guidance overlay shows a target ring and a palm indicator that grows as the hand gets closer, with instructions such as "Open your hand", "Move your palm closer", and "Hold still!".
* Capture triggers automatically once the palm has been correctly aligned for several consecutive frames. There is no shutter button.
* The torch is held on throughout, because palm ridge contrast depends on it.
* Before the shutter fires the SDK waits for autofocus and auto-exposure to settle, so the captured frame is sharp.
* After capture, the image is re-checked for blur, exposure, and palm alignment. If the hand moved at the moment of capture, the user is told and capture resumes automatically rather than failing the call.
* The user can dismiss the screen at any time, which returns `.cancelled`.

## Dependencies

The SDK bundles the following dependency. It is statically linked into the XCFramework, so you do not need to add it to your project:

| Dependency | Version |
|------------|---------|
| ONNX Runtime | 1.24.2 |
