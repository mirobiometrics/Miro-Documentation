# Miro Android SDK Documentation

This documentation outlines how to integrate the [Miro](https://mirobiometrics.com/) Android SDK for palm biometric enrollment, recognition, and deletion. The SDK handles camera capture, palm detection, image encryption, and API communication.

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

## API Methods

All SDK methods are `suspend` functions and must be called from a coroutine scope. Each method launches the camera capture UI, guides the user through palm positioning, and then calls the Miro API.

### Enroll

Creates a biometric profile from a palm image.

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
            // result.profileId  - matched/created profile ID
            // result.customerId - associated customer ID (if set)
            // result.requestId  - unique request identifier
            // result.rawJson    - full JSON response
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
| `requestId` | `String?` | Unique request identifier |
| `error` | `String?` | Error code (on failure) |
| `detail` | `String?` | Error description (on failure) |
| `rawJson` | `String` | Raw JSON response body |

**Error Codes:**

| Error Code | Description |
|------------|-------------|
| `MISSING_CREDENTIALS` | `MiroSDK.configure()` was not called |
| `KEY_EXCHANGE_FAILED` | Failed to fetch RSA public key from server |
| `IMAGE_QUALITY` | Captured image failed quality checks (too blurry, too dark, or too bright) |
| `CAPTURE_CANCELLED` | User dismissed the capture UI |

## Full Example

A complete activity demonstrating all three operations with camera permission handling:

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

## Dependencies

The SDK bundles the following dependencies. These are included transitively when using Maven:

| Dependency | Version |
|------------|---------|
| CameraX | 1.3.1 |
| MediaPipe Tasks Vision | 0.10.14 |
| AndroidX Fragment | 1.6.2 |
| Kotlin Coroutines | 1.7.3 |
