# Webhooks

Learn how to receive and verify webhook notifications from the Grid API.

## Overview

Grid sends webhook events to your configured endpoint to notify you of important events in real-time. Instead of polling for updates, webhooks push events to your server as they happen.

## Configuring Webhooks

Set your webhook URL in platform configuration:

```bash
curl -X PATCH "https://api.lightspark.com/grid/2025-10-13/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://your-domain.com/webhooks/grid"
  }'
```

## Webhook Verification

Grid signs all webhook payloads. Verify signatures to ensure webhooks are authentic:

### Signature Format

```
X-Grid-Signature: sha256=<hex_encoded_signature>
X-Grid-Timestamp: <unix_timestamp>
```

### Verification Code (Node.js)

```javascript
import crypto from 'crypto';

function verifyWebhook(payload, signature, timestamp, secret) {
  // Check timestamp (prevent replay attacks)
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - timestamp) > 300) {
    throw new Error('Webhook timestamp too old');
  }

  // Verify signature
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

### Verification Code (Python)

```python
import hmac
import hashlib
import time

def verify_webhook(payload, signature, timestamp, secret):
    # Check timestamp
    now = int(time.time())
    if abs(now - timestamp) > 300:
        raise ValueError('Webhook timestamp too old')

    # Verify signature
    expected = hmac.new(
        secret.encode(),
        f'{timestamp}.{payload}'.encode(),
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(signature, expected)
```

## Webhook Payload Structure

```json
{
  "eventType": "transaction.status_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "id": "Transaction:...",
    "status": "COMPLETED",
    "previousStatus": "IN_FLIGHT",
    ...
  }
}
```

## Responding to Webhooks

Your endpoint should:
1. Verify the signature
2. Process the event
3. Return a 200 status code quickly

```javascript
app.post('/webhooks/grid', async (req, res) => {
  // Verify signature
  const signature = req.headers['x-grid-signature'];
  const timestamp = req.headers['x-grid-timestamp'];

  if (!verifyWebhook(req.body, signature, timestamp, WEBHOOK_SECRET)) {
    return res.status(401).send('Invalid signature');
  }

  // Process event asynchronously
  processEvent(req.body);

  // Return 200 immediately
  res.status(200).send('OK');
});
```

## Retry Policy

If your endpoint doesn't return 200:
- Grid retries up to 5 times
- Exponential backoff: 1s, 2s, 4s, 8s, 16s
- After 5 failures, the webhook is marked as failed

## Webhook Event Types

| Event Type | Description |
|-----------|-------------|
| `customer.status_change` | Customer status updated |
| `verification.status_change` | KYC/KYB status changed |
| `transaction.status_change` | Transaction status updated |
| `incoming_payment.received` | New incoming payment |
| `internal_account.status_change` | Account balance/status updated |
| `bulk_upload.completed` | CSV bulk upload finished |
| `invitation.claimed` | UMA invitation claimed |
| `agent.action_pending_approval` | Agent action needs approval |
| `card.state_change` | Card state changed |
| `card.funding_source_change` | Card funding source updated |

See [Webhooks section](../13-webhooks/) for detailed documentation on each event.

## Best Practices

1. **Verify signatures** - Always verify webhook authenticity
2. **Return 200 quickly** - Process events asynchronously
3. **Handle duplicates** - Use idempotency keys to prevent duplicate processing
4. **Log events** - Keep a log of all webhook events for debugging
5. **Implement retries** - Handle transient failures gracefully
6. **Use HTTPS** - Webhook URLs must use HTTPS
7. **Monitor delivery** - Check dashboard for failed webhook deliveries

## Testing Webhooks

### Local Development

Use tools like ngrok to test webhooks locally:

```bash
ngrok http 3000
# Use the HTTPS URL as your webhook URL
```

### Sandbox Test Webhook

```bash
curl -X POST "https://api.lightspark.com/grid/2025-10-13/sandbox/webhooks/send-test" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"eventType": "customer.status_change"}'
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Webhooks not received | Check URL is accessible, HTTPS, returns 200 |
| Signature verification fails | Check webhook secret, payload formatting |
| Duplicate events | Implement idempotency checks |
| Events out of order | Use timestamps to order events |
