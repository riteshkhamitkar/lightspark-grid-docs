# Test Webhook

Webhook that is sent once to verify your webhook endpoint is correctly set up.

## Trigger

Sent when you configure or update your platform settings with a webhook URL.

## Event Type

```
test.webhook
```

## Payload

```json
{
  "eventType": "test.webhook",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "message": "This is a test webhook from Grid"
  }
}
```

## Response

Return 200 to confirm your endpoint is working.

## Manual Trigger

```bash
curl -X POST "$GRID_BASE_URL/sandbox/webhooks/send-test" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"eventType": "customer.status_change"}'
```
