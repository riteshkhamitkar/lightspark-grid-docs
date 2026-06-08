# Authentication

Grid API uses HTTP Basic Authentication with your client ID as the username and your client secret as the password.

## Basic Auth

```bash
curl -u "{client_id}:{client_secret}" \
  https://api.lightspark.com/grid/2025-10-13/...
```

## Headers

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Basic <base64(client_id:client_secret)>` | Yes |
| `Content-Type` | `application/json` | For POST/PUT/PATCH |
| `Idempotency-Key` | Unique string | Recommended |
| `Grid-Wallet-Signature` | Base64 signature | For Global Account withdrawals |

## Environment Variables

```bash
export GRID_CLIENT_ID="your_client_id"
export GRID_CLIENT_SECRET="your_client_secret"
```

## API Token Management

### Create API Token
```bash
POST /api-tokens
```

### List API Tokens
```bash
GET /api-tokens
```

### Get API Token
```bash
GET /api-tokens/{id}
```

### Delete API Token
```bash
DELETE /api-tokens/{id}
```

## Errors

| Status | Meaning |
|--------|---------|
| 401 | Invalid or missing credentials |
| 403 | Valid credentials, insufficient permissions |
