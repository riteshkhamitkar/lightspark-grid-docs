# Global P2P Platform Configuration

Configuring credentials, webhooks and currencies for your platform global P2P payments.

## Required Currencies

```bash
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "supportedCurrencies": ["USD", "EUR", "MXN", "BRL", "PHP", "INR", "BTC"]
  }'
```

## UMA Domain Setup

Configure your UMA domain in the Grid dashboard to enable `$user@your-domain.com` addresses.

## Webhook URL

```bash
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://your-domain.com/webhooks/grid"
  }'
```

## Webhook Events

| Event | Description |
|-------|-------------|
| `transaction.status_change` | Payment status updated |
| `incoming_payment.received` | New incoming UMA payment |
| `customer.status_change` | KYC status changes |

## Best Practices

1. **Enable UMA** - Set up UMA domain for instant payments
2. **Multi-currency** - Support popular corridors
3. **Test corridors** - Verify each currency pair works
