# Embedded Wallet Auth API

For Global Accounts authentication.

## Create an Authentication Credential

```bash
POST /auth/credentials
```

```bash
curl -X POST "$GRID_BASE_URL/auth/credentials" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "internalAccountId": "InternalAccount:ga-...",
    "credentialType": "PASSKEY",
    "name": "Device Name"
  }'
```

## List Authentication Credentials

```bash
GET /auth/credentials?internalAccountId=InternalAccount:...
```

## Verify an Authentication Credential

```bash
POST /auth/credentials/{id}/verify
```

## Re-issue an Authentication Credential Challenge

```bash
POST /auth/credentials/{id}/challenge
```

## Revoke an Authentication Credential

```bash
DELETE /auth/credentials/{id}
```

## List Active Sessions

```bash
GET /auth/sessions?internalAccountId=InternalAccount:...
```

## Refresh an Authentication Session

```bash
POST /auth/sessions/{id}/refresh
```

## Revoke an Authentication Session

```bash
DELETE /auth/sessions/{id}
```

## Credential Types

| Type | Description |
|------|-------------|
| `PASSKEY` | WebAuthn/FIDO2 |
| `OAUTH` | OpenID Connect |
| `EMAIL_OTP` | Email one-time password |
