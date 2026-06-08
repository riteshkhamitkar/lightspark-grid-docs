# Client Keys & Signing

Generate the P-256 client key pair, decrypt the session signing key, and sign account actions on Web, iOS, and Android.

## Key Architecture

Global Accounts use ECDSA P-256 keys for signing:
- **Credential Key** - Created during registration (WebAuthn, stored on device)
- **Session Signing Key** - Issued after authentication, used for transaction signing

## Key Generation

### Web (JavaScript)

```javascript
// Generate P-256 key pair
const keyPair = await crypto.subtle.generateKey(
  {
    name: "ECDSA",
    namedCurve: "P-256"
  },
  true, // extractable
  ["sign", "verify"]
);

// Export public key
const publicKey = await crypto.subtle.exportKey("raw", keyPair.publicKey);
const publicKeyBase64 = btoa(String.fromCharCode(...new Uint8Array(publicKey)));
```

### iOS (Swift)

```swift
import CryptoKit

// Generate P-256 private key
let privateKey = P256.KeyAgreement.PrivateKey()
let publicKey = privateKey.publicKey

// Export public key
let publicKeyData = publicKey.rawRepresentation
let publicKeyBase64 = publicKeyData.base64EncodedString()
```

### Android (Kotlin)

```kotlin
import java.security.KeyPairGenerator
import java.security.spec.ECGenParameterSpec

// Generate P-256 key pair
val keyPairGenerator = KeyPairGenerator.getInstance("EC")
keyPairGenerator.initialize(ECGenParameterSpec("secp256r1"))
val keyPair = keyPairGenerator.generateKeyPair()

// Export public key
val publicKey = keyPair.public.encoded
val publicKeyBase64 = Base64.encodeToString(publicKey, Base64.DEFAULT)
```

## Session Signing Key

After authentication, Grid returns an encrypted session signing key:

```json
{
  "sessionId": "Session:...",
  "encryptedSessionSigningKey": "base64_encrypted_key...",
  "expiresAt": "2025-01-15T12:00:00Z"
}
```

### Decrypting the Session Key

The session key is encrypted using HPKE (Hybrid Public Key Encryption) to your client's public key.

```javascript
// Decrypt using credential private key
const sessionSigningKey = await hpkeDecrypt(
  credentialPrivateKey,
  base64ToBuffer(encryptedSessionSigningKey)
);
```

## Signing Transactions

### Get payloadToSign from Quote

When creating a withdrawal quote, the response includes:

```json
{
  "paymentInstructions": {
    "payloadToSign": "base64_payload..."
  }
}
```

### Sign the Payload

```javascript
// Sign with session signing key
const signature = await crypto.subtle.sign(
  { name: "ECDSA", hash: "SHA-256" },
  sessionSigningKey,
  base64ToBuffer(payloadToSign)
);

const base64Signature = btoa(String.fromCharCode(...new Uint8Array(signature)));
```

### Execute Quote with Signature

```bash
curl -X POST "$GRID_BASE_URL/quotes/{quoteId}/execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -H "Grid-Wallet-Signature: $BASE64_SIGNATURE"
```

## Signed-Retry Pattern

Some operations use a signed-retry pattern for security:

1. First attempt may fail with `SIGNATURE_REQUIRED`
2. Sign the payload and retry
3. Grid verifies the signature before proceeding

## Key Management Best Practices

1. **Private keys stay on device** - Never transmit private keys
2. **Secure enclave** - Use hardware security when available
3. **Key rotation** - Support credential rotation
4. **Backup credentials** - Allow multiple credentials per account
5. **Revocation** - Provide way to revoke lost/compromised keys

## Platform Responsibilities

Your platform backend:
1. Creates and manages credentials via Grid API
2. Proxies authentication challenges to client devices
3. Forwards signed transactions to Grid
4. Never stores customer's private keys
