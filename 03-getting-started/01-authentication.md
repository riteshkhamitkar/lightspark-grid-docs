# Authentication

Grid API uses HTTP Basic Authentication with your client ID as the username and your client secret as the password.

## Basic Auth

```bash
curl -u "{client_id}:{client_secret}" \
  https://api.lightspark.com/grid/2025-10-13/...
```

## Getting Credentials

1. Log in to the [Grid Dashboard](https://app.lightspark.com)
2. Navigate to API Keys section
3. Generate new sandbox or production keys
4. Copy the Client ID and Client Secret

## Environment Variables

```bash
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
export GRID_CLIENT_ID="YOUR_CLIENT_ID"
export GRID_CLIENT_SECRET="YOUR_CLIENT_SECRET"
```

## Using with cURL

```bash
curl -X GET "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json"
```

## Using with SDKs

If integrating with one of our SDKs, the SDK reads environment variables and will populate the credentials for you.

### Node.js
```javascript
import { GridClient } from '@lightsparkdev/grid-sdk';

const grid = new GridClient();
// Automatically reads GRID_CLIENT_ID and GRID_CLIENT_SECRET
```

### Python
```python
import grid_sdk

grid = grid_sdk.GridClient()
# Automatically reads GRID_CLIENT_ID and GRID_CLIENT_SECRET
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

### Delete API Token
```bash
DELETE /api-tokens/{id}
```

## Security Best Practices

1. **Never expose credentials client-side** - Only use credentials on your backend
2. **Use environment variables** - Don't hardcode credentials
3. **Rotate keys regularly** - Delete old keys and create new ones
4. **Use separate keys per environment** - Different keys for sandbox and production
5. **Monitor API usage** - Check dashboard for unexpected usage patterns
6. **Restrict IP addresses** - Contact Lightspark for IP whitelisting

## Authorization Errors

| Status | Error | Description |
|--------|-------|-------------|
| 401 | Unauthorized | Invalid or missing credentials |
| 403 | Forbidden | Valid credentials but insufficient permissions |

## Multi-Factor Authentication

For dashboard access, enable MFA in your account settings for additional security.
