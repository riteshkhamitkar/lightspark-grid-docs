# Platform Configuration

Configuring credentials, webhooks and currencies for your Payouts platform.

## Required Configuration

### Supported Currencies

Configure currencies you'll use for payouts:

```bash
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "supportedCurrencies": ["USD", "MXN", "BRL", "PHP", "EUR"]
  }'
```

### Webhook URL

Set up webhook endpoint for payment notifications:

```bash
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://your-domain.com/webhooks/grid"
  }'
```

## Recommended Currency Setup

### For US-to-World Payouts
```json
["USD", "MXN", "BRL", "PHP", "INR", "EUR", "GBP"]
```

### For EU-based Payouts
```json
["EUR", "USD", "GBP", "MXN", "BRL", "PHP"]
```

### For Global Operations
```json
["USD", "EUR", "MXN", "BRL", "PHP", "INR", "GBP", "CNY", "JPY"]
```

## Webhook Events to Handle

| Event | Description | Action |
|-------|-------------|--------|
| `customer.status_change` | KYC status updated | Enable/disable payments |
| `transaction.status_change` | Payment status changed | Update payment record |
| `incoming_payment.received` | Funds received | Update balance |
| `internal_account.status_change` | Account updated | Sync balance |

## Testing Configuration

### Sandbox Setup

1. Generate sandbox credentials
2. Configure test webhook URL (ngrok for local dev)
3. Set test currencies
4. Test with magic values

### Production Setup

1. Request production access from Lightspark
2. Configure production webhook URL
3. Set production currencies
4. Monitor closely after launch

## Dashboard Settings

Additional settings available in the Grid Dashboard:

- **Branding** - Logo, colors for KYC flow
- **Compliance** - KYC provider settings
- **Notifications** - Email notification preferences
- **Team** - User access management
- **API Keys** - Credential rotation

## Best Practices

1. **Start with fewer currencies** - Add more as needed
2. **Test webhooks thoroughly** - Before going live
3. **Monitor dashboard** - Regular check of settings
4. **Rotate credentials** - Periodic key rotation
5. **Document configuration** - Keep internal documentation
