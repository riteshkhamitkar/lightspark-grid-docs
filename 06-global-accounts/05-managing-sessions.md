# Managing Sessions

List and revoke active sessions on a Global Account.

## What is a Session?

A session is created each time a credential is verified via `POST /auth/credentials/{id}/verify`. It remains active until its `expiresAt` passes or it is revoked via `DELETE /auth/sessions/{id}`.

## List Active Sessions

```bash
curl -X GET "$GRID_BASE_URL/auth/sessions?internalAccountId=InternalAccount:ga-abc123" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

### Response

```json
{
  "data": [
    {
      "id": "Session:session-1",
      "credentialId": "Credential:passkey-1",
      "credentialType": "PASSKEY",
      "deviceInfo": "iPhone 15 - Safari",
      "createdAt": "2025-01-15T10:00:00Z",
      "expiresAt": "2025-01-15T12:00:00Z",
      "lastUsedAt": "2025-01-15T11:30:00Z"
    },
    {
      "id": "Session:session-2",
      "credentialId": "Credential:otp-1",
      "credentialType": "EMAIL_OTP",
      "deviceInfo": "Chrome on MacOS",
      "createdAt": "2025-01-15T09:00:00Z",
      "expiresAt": "2025-01-15T11:00:00Z",
      "lastUsedAt": "2025-01-15T10:45:00Z"
    }
  ]
}
```

## Session Fields

| Field | Description |
|-------|-------------|
| `id` | Session unique ID |
| `credentialId` | Associated credential |
| `credentialType` | PASSKEY, OAUTH, EMAIL_OTP |
| `deviceInfo` | Device/browser information |
| `createdAt` | Session creation time |
| `expiresAt` | Session expiration time |
| `lastUsedAt` | Last activity timestamp |

## Revoke a Session

```bash
curl -X DELETE "$GRID_BASE_URL/auth/sessions/Session:session-1" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Revocation is immediate. Any in-progress operations using this session will fail.

## Revocation Signed-Retry

Session revocation uses a two-step signed-retry flow:

1. Initiate revocation
2. Sign the revocation request
3. Complete revocation

This prevents unauthorized session termination.

## Refresh a Session

```bash
curl -X POST "$GRID_BASE_URL/auth/sessions/Session:session-1/refresh" \
  -H "Content-Type: application/json" \
  -d '{
    "clientPublicKey": "base64_public_key..."
  }'
```

Returns a new session signing key, extending the session lifetime.

## Best Practices

1. **Show active sessions** - Let users see where they're logged in
2. **Allow remote logout** - Let users revoke other sessions
3. **Auto-expire** - Sessions expire automatically after inactivity
4. **Refresh proactively** - Refresh before expiration
5. **Monitor usage** - Track session patterns for security
