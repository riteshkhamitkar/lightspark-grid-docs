# Ramps Platform Configuration

Configure your platform for ramp operations.

## Required Currencies

```bash
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "supportedCurrencies": ["USD", "BTC", "USDC", "USDB"]
  }'
```

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
| `transaction.status_change` | Conversion status |
| `incoming_payment.received` | Funds deposited |
| `internal_account.status_change` | Balance updates |

## Best Practices

1. **Enable all needed currencies** upfront
2. **Test conversion rates** - Display to users
3. **Monitor slippage** - Rate changes during execution
4. **Set limits** - Min/max conversion amounts
5. **Compliance** - Ensure KYC before crypto transactions
