# Platform Configuration for Rewards

Configuring platform settings for rewards distribution.

## Required Currencies

```bash
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "supportedCurrencies": ["USD", "BTC"]
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

## Dashboard Settings

- **Branding** - Logo for any KYC flows
- **Notifications** - Reward completion emails
- **Limits** - Per-recipient reward limits
