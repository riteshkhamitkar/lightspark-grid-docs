# Authentication

Register, reauthenticate, and manage Global Account credentials (passkey, OAuth, email OTP).

## Credential Types

| Type | Description | Best For |
|------|-------------|----------|
| `PASSKEY` | WebAuthn/FIDO2 biometric | Mobile apps, modern browsers |
| `OAUTH` | OpenID Connect (Google, Apple) | Social login |
| `EMAIL_OTP` | One-time password via email | Fallback, simple setup |

## Registration Flow

### 1. Create Credential

```bash
POST /auth/credentials
```

```bash
curl -X POST "$GRID_BASE_URL/auth/credentials" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "internalAccountId": "InternalAccount:ga-abc123",
    "credentialType": "PASSKEY",
    "name": "Jane\'s iPhone"
  }'
```

### 2. Complete Registration

For **PASSKEY**: Complete WebAuthn ceremony
For **OAUTH**: Redirect to OAuth provider
For **EMAIL_OTP**: Verify OTP code

## Passkey (WebAuthn)

### Registration

```javascript
// 1. Create credential on server
const { challenge, userId } = await server.createCredential(accountId, 'PASSKEY');

// 2. Create passkey on device
const credential = await navigator.credentials.create({
  publicKey: {
    challenge: base64ToBuffer(challenge),
    rp: { name: "Your App", id: window.location.hostname },
    user: { id: base64ToBuffer(userId), name: "user@example.com", displayName: "User" },
    pubKeyCredParams: [{ alg: -7, type: "public-key" }],
    authenticatorSelection: { authenticatorAttachment: "platform" },
    timeout: 60000
  }
});

// 3. Send credential to server
await server.verifyCredential(credential);
```

### Authentication

```javascript
// 1. Get challenge from server
const { challenge, allowCredentials } = await server.startAuth(accountId);

// 2. Use passkey
const assertion = await navigator.credentials.get({
  publicKey: {
    challenge: base64ToBuffer(challenge),
    allowCredentials: allowCredentials.map(id => ({
      id: base64ToBuffer(id),
      type: 'public-key'
    })),
    timeout: 60000
  }
});

// 3. Send assertion to server
const session = await server.completeAuth(assertion);
```

## OAuth (OIDC)

### Configuration

Set up OAuth providers in your platform settings:
- Google
- Apple
- Custom OIDC provider

### Flow

1. Redirect to OAuth provider authorization URL
2. User authenticates with provider
3. Provider redirects back with authorization code
4. Exchange code for tokens
5. Send ID token to Grid to complete registration

## Email OTP

### Registration

```bash
# 1. Create OTP credential
curl -X POST "$GRID_BASE_URL/auth/credentials" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "internalAccountId": "InternalAccount:ga-abc123",
    "credentialType": "EMAIL_OTP",
    "name": "jane@example.com"
  }'

# 2. OTP sent to customer's email
# 3. Verify OTP
curl -X POST "$GRID_BASE_URL/auth/credentials/{credentialId}/verify" \
  -H "Content-Type: application/json" \
  -d '{
    "otpCode": "123456"
  }'
```

### Sandbox OTP

In sandbox, OTP code is always: `000000`

## Listing Credentials

```bash
curl -X GET "$GRID_BASE_URL/auth/credentials?internalAccountId=InternalAccount:ga-abc123" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Revoking Credentials

```bash
curl -X DELETE "$GRID_BASE_URL/auth/credentials/{credentialId}" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Re-authentication

Customers need to re-authenticate for each withdrawal:

1. Start auth with credential
2. Complete auth challenge (biometric, OTP, etc.)
3. Receive session token
4. Use session to sign transaction

## Best Practices

1. **Recommend passkeys** - Most secure and convenient
2. **Offer fallback** - Email OTP as backup
3. **Multiple credentials** - Allow multiple devices
4. **Revoke lost devices** - Provide credential management
5. **Clear instructions** - Guide users through setup
