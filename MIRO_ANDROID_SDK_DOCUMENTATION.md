# Miro Android SDK Documentation

This documentation outlines how to integrate the [Miro](https://mirobiometrics.com/) Android SDK for palm biometric enrollment, recognition, verification, and deletion. The SDK handles camera capture, palm detection, image encryption, and API communication.

## Assumptions

* This documentation assumes you have set up a Miro Identity Instance on the [Miro Admin Dashboard](https://dashboard.mirobiometrics.com/).
* You have your Instance ID and Secret from the dashboard.
* You have downloaded the SDK ZIP from the Miro Admin Dashboard.

## Requirements

| Requirement | Value |
|-------------|-------|
| Min SDK | 24 (Android 7.0) |
| Language | Kotlin |
| Activity | Must extend `FragmentActivity` (or `AppCompatActivity`) |
| Permission | `android.permission.CAMERA` (must be granted before calling SDK) |

## Installation

Add the Miro SDK dependency to your app's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.miro:palm-sdk:1.0.0")
}
```

Replace `1.0.0` with the version in the ZIP you downloaded — releases are versioned `1.0.<build>`, and the exact value is visible in the Maven path inside the archive.

If using a local Maven repository, add `mavenLocal()` to your `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        mavenLocal()
        google()
        mavenCentral()
    }
}
```

## Configuration

Configure the SDK with your credentials before making any API calls. This should be done once, typically when the user provides or your app loads the stored credentials:

```kotlin
MiroSDK.configure(MiroSDK.Credentials(
    instanceId = "your-instance-id",
    secret = "your-secret-key"
))
```

### Capture options

Pass a `CaptureConfig` to change which lens is used:

```kotlin
MiroSDK.configure(
    credentials = MiroSDK.Credentials(instanceId = "...", secret = "..."),
    captureConfig = MiroSDK.CaptureConfig(
        cameraSelector = CameraSelector.DEFAULT_FRONT_CAMERA,
        imageMirrored = true
    )
)
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `cameraSelector` | `CameraSelector` | `DEFAULT_BACK_CAMERA` | Which lens to bind. The back lens is strongly preferred — it has a torch, and palm ridge contrast depends on it. |
| `imageMirrored` | `Boolean` | `false` | Set `true` when the lens produces horizontally mirrored output, so the backend un-mirrors before matching. Required for the front (selfie) camera. |

## API Methods

All SDK methods are `suspend` functions and must be called from a coroutine scope. Each method launches the camera capture UI, guides the user through palm positioning, and then calls the Miro API.

### Enroll

Creates a biometric profile from a palm image. Captures the first palm, then offers a second — enrolling both hands materially improves later recognition, so the prompt is shown either way.

```kotlin
lifecycleScope.launch {
    val result = MiroSDK.enroll(
        activity = this@MainActivity,
        customerId = "user-123",       // optional
        customerData = "encrypted-data" // optional
    )
    handleResult(result)
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `activity` | `FragmentActivity` | Yes | The hosting activity |
| `customerId` | `String?` | No | Unique customer identifier to associate with the profile |
| `customerData` | `String?` | No | Encrypted customer data to store with the profile |
| `requireTwoPalms` | `Boolean` | No | When `true`, the second palm cannot be skipped. Defaults to `false` |

### Recognize

Matches a palm against enrolled profiles in your Identity Instance.

```kotlin
lifecycleScope.launch {
    val result = MiroSDK.recognize(this@MainActivity)
    handleResult(result)
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `activity` | `FragmentActivity` | Yes | The hosting activity |
| `confidenceTiers` | `Boolean` | No | When `true`, the backend grades the match into `HIGH`/`MEDIUM`/`LOW` and returns it in `confidenceTier`. Defaults to `false` |

### Verify

Checks a palm against one claimed identity, rather than searching every profile.

```kotlin
lifecycleScope.launch {
    val result = MiroSDK.verify(
        activity = this@MainActivity,
        type = MiroSDK.VerificationType.CUSTOMER_ID,
        id = "user-123"
    )
    handleResult(result)
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `activity` | `FragmentActivity` | Yes | The hosting activity |
| `type` | `VerificationType` | Yes | `PROFILE_ID` or `CUSTOMER_ID` — which identifier `id` refers to |
| `id` | `String` | Yes | The identifier to verify the palm against |

A completed check that does not match returns `FAILURE` with error `VERIFICATION_FAILED`. That is a normal, expected answer rather than a fault; branch on the error code, not on the result type alone.

### Delete

Removes a profile by palm match from your Identity Instance.

```kotlin
lifecycleScope.launch {
    val result = MiroSDK.delete(this@MainActivity)
    handleResult(result)
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `activity` | `FragmentActivity` | Yes | The hosting activity |

## Handling Results

All methods return an `MiroSDK.SdkResult`:

```kotlin
fun handleResult(result: MiroSDK.SdkResult) {
    when (result.type) {
        MiroSDK.ResultType.SUCCESS -> {
            // result.profileId      - matched/created profile ID
            // result.customerId     - associated customer ID (if set)
            // result.requestId      - unique request identifier
            // result.confidenceTier - match grade, if confidenceTiers was requested
        }
        MiroSDK.ResultType.FAILURE -> {
            // result.error  - error code (e.g. "IMAGE_QUALITY", "MISSING_CREDENTIALS")
            // result.detail - human-readable description
        }
        MiroSDK.ResultType.CANCELLED -> {
            // User dismissed the capture UI
        }
    }
}
```

**Result Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `type` | `ResultType` | `SUCCESS`, `FAILURE`, or `CANCELLED` |
| `profileId` | `String?` | The biometric profile ID (on success) |
| `customerId` | `String?` | Customer ID associated with the profile |
| `customerData` | `String?` | Customer data associated with the profile |
| `requestId` | `String?` | Unique request identifier. Quote this when raising a support issue |
| `error` | `String?` | Error code (on failure) |
| `detail` | `String?` | Error description (on failure) |
| `confidenceTier` | `ConfidenceTier?` | `HIGH`, `MEDIUM`, or `LOW`, when `confidenceTiers` was requested |
| `toJson()` | `String` | Pretty-printed JSON of the result, for logging |

**Error Codes:**

| Error Code | Description |
|------------|-------------|
| `MISSING_CREDENTIALS` | `MiroSDK.configure()` was not called |
| `KEY_EXCHANGE_FAILED` | Failed to fetch RSA public key from server |
| `IMAGE_QUALITY` | Captured image failed quality checks (too blurry, too dark, or too bright) |
| `CAPTURE_CANCELLED` | User dismissed the capture UI |
| `VERIFICATION_FAILED` | The verification check completed and the palm is not the claimed identity |
| `SDK_UPDATE_REQUIRED` | The backend no longer supports this SDK version — ship an update |
| `UNKNOWN_ERROR` | The server returned an error the SDK could not classify |

## Full Example

A complete activity demonstrating all four operations with camera permission handling:

```kotlin
class MainActivity : FragmentActivity() {

    private var pendingAction: (() -> Unit)? = null

    private val cameraPermission = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) pendingAction?.invoke()
        else Toast.makeText(this, "Camera permission required", Toast.LENGTH_SHORT).show()
        pendingAction = null
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        MiroSDK.configure(MiroSDK.Credentials(
            instanceId = "<instance-id>",
            secret = "<secret>"
        ))

        findViewById<View>(R.id.enrollBtn).setOnClickListener {
            requireCamera {
                lifecycleScope.launch {
                    val result = MiroSDK.enroll(
                        this@MainActivity,
                        customerId = "user-123"
                    )
                    handleResult(result)
                }
            }
        }

        findViewById<View>(R.id.recognizeBtn).setOnClickListener {
            requireCamera {
                lifecycleScope.launch {
                    handleResult(MiroSDK.recognize(this@MainActivity))
                }
            }
        }

        findViewById<View>(R.id.verifyBtn).setOnClickListener {
            requireCamera {
                lifecycleScope.launch {
                    handleResult(MiroSDK.verify(
                        this@MainActivity,
                        type = MiroSDK.VerificationType.CUSTOMER_ID,
                        id = "user-123"
                    ))
                }
            }
        }

        findViewById<View>(R.id.deleteBtn).setOnClickListener {
            requireCamera {
                lifecycleScope.launch {
                    handleResult(MiroSDK.delete(this@MainActivity))
                }
            }
        }
    }

    private fun requireCamera(action: () -> Unit) {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
            == PackageManager.PERMISSION_GRANTED) {
            action()
        } else {
            pendingAction = action
            cameraPermission.launch(Manifest.permission.CAMERA)
        }
    }

    private fun handleResult(result: MiroSDK.SdkResult) {
        when (result.type) {
            MiroSDK.ResultType.SUCCESS -> {
                Log.d("Miro", "Profile: ${result.profileId}")
            }
            MiroSDK.ResultType.FAILURE -> {
                Log.e("Miro", "Error: ${result.error} - ${result.detail}")
            }
            MiroSDK.ResultType.CANCELLED -> {
                Log.d("Miro", "Cancelled")
            }
        }
    }
}
```

## Capture Behavior

The capture UI is added over your activity's content and locked to portrait for its lifetime. It handles the whole capture loop for you:

* A guidance overlay shows a target ring and a palm indicator that grows as the hand gets closer, with instructions such as "Open your hand", "Move your palm closer", and "Hold still!".
* Capture triggers automatically once the palm has been correctly aligned for several consecutive frames. There is no shutter button.
* The torch is held on throughout, because palm ridge contrast depends on it.
* After capture, the image is re-checked for blur, exposure, and palm alignment. If the hand moved at the moment of capture, the user is told and capture resumes automatically rather than failing the call.
* The user can dismiss the UI at any time, which returns `CANCELLED`.

## Dependencies

The SDK bundles the following dependencies. These are included transitively when using Maven:

| Dependency | Version |
|------------|---------|
| CameraX | 1.3.1 |
| ONNX Runtime Android | 1.23.2 |
| AndroidX Fragment | 1.6.2 |
| Kotlin Coroutines | 1.7.3 |
