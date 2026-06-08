# Sandbox API

Endpoints for testing in sandbox environment.

## Simulate Funding an Internal Account

```bash
POST /sandbox/internal-accounts/{id}/fund
```

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:.../fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "1000.00",
    "currency": "USD"
  }'
```

## Simulate Sending Funds

```bash
POST /sandbox/simulate/send
```

## Simulate Payment Send to Test Receiving UMA Payment

```bash
POST /sandbox/uma/payments/simulate
```

## Simulate a Card Authorization

```bash
POST /sandbox/cards/{card_id}/authorize
```

## Simulate a Card Clearing

```bash
POST /sandbox/cards/{card_id}/clear
```

## Simulate a Card Return

```bash
POST /sandbox/cards/{card_id}/return
```

## Send a Test Webhook

```bash
POST /sandbox/webhooks/send-test
```

```bash
curl -X POST "$GRID_BASE_URL/sandbox/webhooks/send-test" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "customer.status_change"
  }'
```

## Sandbox Magic Values

See [Sandbox Testing](../../03-getting-started/03-sandbox-testing.md) for all magic values.
