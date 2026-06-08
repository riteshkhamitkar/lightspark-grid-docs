# Webhooks Overview

Learn how to receive and verify webhook notifications from the Grid API.

## What are Webhooks?

Webhooks are HTTP callbacks that Grid sends to your server when events occur. Instead of polling for updates, webhooks push events to your endpoint in real-time.

## Configuring Webhooks

Set your webhook URL in platform configuration:

```bash
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://your-domain.com/webhooks/grid"
  }'
```

## Webhook Verification

Grid signs all webhook payloads. Verify signatures to ensure authenticity.

### Signature Headers

```
X-Grid-Signature: sha256=<hex_signature>
X-Grid-Timestamp: <unix_timestamp>
```

### Verification (Node.js)

```javascript
import crypto from 'crypto';

function verifyWebhook(payload, signature, timestamp, secret) {
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - timestamp) > 300) {
    throw new Error('Webhook timestamp too old');
  }

  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expected, 'hex')
  );
}
```

### Verification (Python)

```python
import hmac
import hashlib
import time

def verify_webhook(payload, signature, timestamp, secret):
    now = int(time.time())
    if abs(now - timestamp) > 300:
        raise ValueError('Webhook timestamp too old')

    expected = hmac.new(
        secret.encode(),
        f'{timestamp}.{payload}'.encode(),
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(signature, expected)
```

## Responding to Webhooks

Your endpoint should:
1. Verify the signature
2. Process the event asynchronously
3. Return a 200 status code quickly

## Retry Policy

If your endpoint doesn't return 200:
- Grid retries up to 5 times
- Exponential backoff: 1s, 2s, 4s, 8s, 16s
- After 5 failures, the webhook is marked as failed

## All Webhook Event Types

| Event Type | Description |
|-----------|-------------|
| `customer.status_change` | Customer status updated |
| `verification.status_change` | KYC/KYB status changed |
| `transaction.status_change` | Transaction status updated |
| `incoming_payment.received` | Incoming payment received |
| `internal_account.status_change` | Account balance/status changed |
| `bulk_upload.status_change` | CSV bulk upload completed |
| `invitation.claimed` | UMA invitation claimed |
| `agent.action_pending_approval` | Agent action needs approval |
| `card.state_change` | Card state changed |
| `card.funding_source_change` | Card funding source updated |

## Payload Structure

```json
{
  "eventType": "transaction.status_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": { ... }
}
```

## Best Practices

1. **Verify signatures** - Always verify authenticity
2. **Return 200 quickly** - Process asynchronously
3. **Handle duplicates** - Use idempotency checks
4. **Log events** - Keep audit trail
5. **Implement retries** - Handle transient failures
6. **Use HTTPS** - Required for webhook URLs
7. **Monitor delivery** - Check dashboard for failures
