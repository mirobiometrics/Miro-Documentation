# Miro API Documentation

This documentation outlines how to call the [Miro](https://mirobiometrics.com/) APIs for profile enrollment, recognition, and deletion.

A client package for Node that follows these steps can be found [here](https://github.com/mirobiometrics/Miro-JSClient).

## Assumptions
* This documentation assumes you have set up a Miro Identity Instance on the [Miro Admin Dashboard](https://dashboard.mirobiometrics.com/).
* This examples in this documentation uses the the command line and Python CLI in example to process images. The examples in this documentation require:
  *  Python 3.7+
  *  `cryptography` package: `pip3 install cryptography`
  *  `curl` command-line tool

## Requirements
* Image Requirements:
  * Images must be less than 6Mbs in size.
  * Image resolution must be between 800x800 and 2400x2400.
* Request Requirements:
  * Request timestamps must be must be within a few minutes of the current server time.
  * Each request must contain a unique image. Submitting the same image multiple times will be rejected.

## API Authentication

All requests require HMAC-SHA256 signature authentication.

**Headers (all endpoints):**
| Header | Description |
|--------|-------------|
| `X-Instance-Id` | Your instance ID |
| `X-Timestamp` | Unix timestamp (seconds) |
| `X-Signature` | HMAC-SHA256 signature (base64) |

**Signature Format:**
```
HMAC-SHA256(secret, "{METHOD}\n{PATH}\n{TIMESTAMP}\n{METADATA}")
```

## Image Encryption

All palm images must be encrypted before transmission.

### Step 1: Fetch RSA Public Key
```
GET /api/key-exchange/rsa
```
Returns PEM-encoded RSA public key.

```bash
curl https://orchestrator.mirobiometrics.com/api/key-exchange/rsa -o rsa_public.pem
```

### Step 2: Encrypt Image

**Generate AES-256 key and 12-byte IV:**
```bash
python3 << 'EOF'
import os
key = os.urandom(32)  # 256 bits
iv = os.urandom(12)   # 96 bits
open('aes_key.bin', 'wb').write(key)
open('iv.bin', 'wb').write(iv)
print(f"Generated AES key: {len(key)} bytes")
print(f"Generated IV: {len(iv)} bytes")
EOF
```

**Encrypt image with AES-256-GCM:**
```bash
python3 << 'EOF'
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

image_file = "palm.png"

with open('aes_key.bin', 'rb') as f:
    key = f.read()
with open('iv.bin', 'rb') as f:
    iv = f.read()
with open(image_file, 'rb') as f:
    image_data = f.read()

aesgcm = AESGCM(key)
encrypted = aesgcm.encrypt(iv, image_data, None)
open('encrypted_image.bin', 'wb').write(encrypted)
print(f"Encrypted {len(image_data)} bytes")
EOF
```

**Encrypt AES key with RSA-OAEP:**
```bash
openssl pkeyutl -encrypt -pubin -inkey rsa_public.pem \
  -pkeyopt rsa_padding_mode:oaep \
  -pkeyopt rsa_oaep_md:sha256 \
  -in aes_key.bin > encrypted_key.bin
```

**Base64 encode IV and encrypted image:**
```bash
# macOS
base64 -i iv.bin -o iv_b64.txt
base64 -i encrypted_image.bin -o encrypted_image_b64.txt

# Linux
base64 -w 0 iv.bin > iv_b64.txt
base64 -w 0 encrypted_image.bin > encrypted_image_b64.txt

echo "X-Encrypted-Key: $(cat encrypted_key.txt)"
echo "X-IV: $(cat iv_b64.txt)"
```


## API Calls

### Enroll

Creates a biometric profile from palm image(s) in your Identity Instance.

```
POST /api/enroll
Content-Type: application/json
```

**Request Body:**
```json
{
  "palm1": {
    "encryptedKey": "<base64>",
    "iv": "<base64>",
    "data": "<base64>"
  },
  "palm2": { ... },
  "customerId": "optional-id",
  "customerData": "optional-encrypted-data",
  "imageMirrored": false
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `palm1` | object | Yes | Encrypted palm image |
| `palm2` | object | No | Second encrypted palm image (must be opposite hand) |
| `customerId` | string | No | Unique customer identifier |
| `customerData` | string | No | Encrypted customer data |
| `imageMirrored` | boolean | No | Set to `true` if images are horizontally mirrored (e.g., front-facing camera). Default: `false` |

**Signature Metadata:**
```
{palm1.encryptedKey}:{palm1.iv}:{palm2.encryptedKey}:{palm2.iv}:{customerId}:{customerData}
```
Use empty strings for omitted fields.

**Response:**
```json
{ "profileId": "...", "customerId": "...", "customerData": "...", "requestId": "..." }
```

**Example:**

Assuming you've followed the Image Encryption steps above, calculate the HMAC signature and send the enrollment request:

```bash
# Set your credentials
INSTANCE_ID="your-instance-id"
SECRET="your-secret-key"
TIMESTAMP=$(date +%s)

# Build metadata string (format: encryptedKey:iv:encryptedKey2:iv2:customerId:customerData)
METADATA="$(cat encrypted_key.txt):$(cat iv_b64.txt):::user-123:"

# Calculate HMAC-SHA256 signature
SIGNATURE_STRING="POST\n/api/enroll\n${TIMESTAMP}\n${METADATA}"
SIGNATURE=$(echo -n -e "$SIGNATURE_STRING" | openssl dgst -sha256 -hmac "$SECRET" -binary | base64)

# Send enrollment request
curl -X POST https://orchestrator.mirobiometrics.com/api/enroll \
  -H "Content-Type: application/json" \
  -H "X-Instance-Id: $INSTANCE_ID" \
  -H "X-Timestamp: $TIMESTAMP" \
  -H "X-Signature: $SIGNATURE" \
  -d "{
    \"palm1\": {
      \"encryptedKey\": \"$(cat encrypted_key.txt)\",
      \"iv\": \"$(cat iv_b64.txt)\",
      \"data\": \"$(cat encrypted_image_b64.txt)\"
    },
    \"customerId\": \"user-123\"
  }"
```

### Recognize

Matches a palm against enrolled profiles in your Identity Instance.

```
POST /api/recognize
Content-Type: application/octet-stream
X-Encrypted-Key: <base64>
X-IV: <base64>
X-Image-Mirrored: <true|false>
Body: <encrypted image bytes>
```

| Header | Required | Description |
|--------|----------|-------------|
| `X-Encrypted-Key` | Yes | Base64-encoded RSA-encrypted AES key |
| `X-IV` | Yes | Base64-encoded initialization vector |
| `X-Image-Mirrored` | No | Set to `true` if image is horizontally mirrored (e.g., front-facing camera). Default: `false` |

**Signature Metadata:**
```
{encryptedKey}:{iv}
```

**Response:**
```json
{ "profileId": "...", "customerId": "...", "customerData": "...", "requestId": "..." }
```

**Example:**

Assuming you've followed the Image Encryption steps above, calculate the HMAC signature and send the recognition request:

```bash
# Set your credentials
INSTANCE_ID="your-instance-id"
SECRET="your-secret-key"
TIMESTAMP=$(date +%s)

# Build metadata string (format: encryptedKey:iv)
METADATA="$(cat encrypted_key.txt):$(cat iv_b64.txt)"

# Calculate HMAC-SHA256 signature
SIGNATURE_STRING="POST\n/api/recognize\n${TIMESTAMP}\n${METADATA}"
SIGNATURE=$(echo -n -e "$SIGNATURE_STRING" | openssl dgst -sha256 -hmac "$SECRET" -binary | base64)

# Send recognition request
curl -X POST https://orchestrator.mirobiometrics.com/api/recognize \
  -H "Content-Type: application/octet-stream" \
  -H "X-Instance-Id: $INSTANCE_ID" \
  -H "X-Timestamp: $TIMESTAMP" \
  -H "X-Signature: $SIGNATURE" \
  -H "X-Encrypted-Key: $(cat encrypted_key.txt)" \
  -H "X-IV: $(cat iv_b64.txt)" \
  --data-binary @encrypted_image.bin
```

### Delete

Removes a profile by palm match from your Identity Instance.

```
POST /api/delete
Content-Type: application/octet-stream
X-Encrypted-Key: <base64>
X-IV: <base64>
X-Image-Mirrored: <true|false>
Body: <encrypted image bytes>
```

| Header | Required | Description |
|--------|----------|-------------|
| `X-Encrypted-Key` | Yes | Base64-encoded RSA-encrypted AES key |
| `X-IV` | Yes | Base64-encoded initialization vector |
| `X-Image-Mirrored` | No | Set to `true` if image is horizontally mirrored (e.g., front-facing camera). Default: `false` |

**Signature Metadata:**
```
{encryptedKey}:{iv}
```

**Response:**
```json
{ "profileId": "...", "customerId": "...", "customerData": "...", "requestId": "..." }
```

**Example:**

Assuming you've followed the Image Encryption steps above, calculate the HMAC signature and send the delete request:

```bash
# Set your credentials
INSTANCE_ID="your-instance-id"
SECRET="your-secret-key"
TIMESTAMP=$(date +%s)

# Build metadata string (format: encryptedKey:iv)
METADATA="$(cat encrypted_key.txt):$(cat iv_b64.txt)"

# Calculate HMAC-SHA256 signature
SIGNATURE_STRING="POST\n/api/delete\n${TIMESTAMP}\n${METADATA}"
SIGNATURE=$(echo -n -e "$SIGNATURE_STRING" | openssl dgst -sha256 -hmac "$SECRET" -binary | base64)

curl -X POST https://orchestrator.mirobiometrics.com/api/delete \
  -H "Content-Type: application/octet-stream" \
  -H "X-Instance-Id: $INSTANCE_ID" \
  -H "X-Timestamp: $TIMESTAMP" \
  -H "X-Signature: $SIGNATURE" \
  -H "X-Encrypted-Key: $(cat encrypted_key.txt)" \
  -H "X-IV: $(cat iv_b64.txt)" \
  --data-binary @encrypted_image.bin
```