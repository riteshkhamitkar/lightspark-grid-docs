# Configuration

Configure your Grid integration.

## Platform Configuration

Your platform settings control how your Grid integration behaves.

### Getting Current Configuration

```bash
curl -X GET "https://api.lightspark.com/grid/2025-10-13/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

### Updating Configuration

```bash
curl -X PATCH "https://api.lightspark.com/grid/2025-10-13/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "supportedCurrencies": ["USD", "EUR", "BTC", "USDC", "USDB"],
    "webhookUrl": "https://your-domain.com/webhooks/grid"
  }'
```

## Configuration Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `supportedCurrencies` | string[] | Currencies your platform supports |
| `webhookUrl` | string | URL for webhook notifications |
| `name` | string | Platform name |
| `logoUrl` | string | URL to your platform logo |

## Setting Up Webhooks

1. Create an HTTPS endpoint on your server
2. Update platform configuration with the webhook URL
3. Grid sends a test webhook to verify
4. Start receiving event notifications

## Supported Currencies Configuration

When you configure supported currencies:
- Internal accounts are auto-created for each currency
- Customers created on your platform get accounts for these currencies
- Quotes can be created for any supported currency pair

### Recommended Currency Setup

**For Payouts Platforms:**
```json
["USD", "EUR", "MXN", "BRL", "PHP"]
```

**For Crypto Platforms:**
```json
["USD", "BTC", "USDC", "USDB"]
```

**For Global Operations:**
```json
["USD", "EUR", "GBP", "MXN", "BRL", "PHP", "INR", "BTC", "USDC", "USDB"]
```

## Sandbox vs Production

### Sandbox Environment
- Test all flows without real money
- Use magic values to trigger responses
- Customers auto-approved for KYC
- Instant settlement simulation
- Free to use

### Production Environment
- Real money movement
- Full KYC/KYB verification required
- Actual settlement times
- Requires approval from Lightspark

## API Version

Current API version: `2025-10-13`

All endpoints are prefixed with:
```
https://api.lightspark.com/grid/2025-10-13
```

## Security Configuration

### IP Whitelisting
Contact Lightspark to configure IP whitelisting for production API access.

### Webhook Verification
Verify webhook signatures to ensure they come from Grid:

```python
import hmac
import hashlib

def verify_webhook(payload, signature, secret):
    expected = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

## Dashboard Settings

Configure additional settings in the Grid Dashboard:
- Branding (logo, colors)
- KYC/KYB provider settings
- Compliance settings
- Team member access
- API key management
