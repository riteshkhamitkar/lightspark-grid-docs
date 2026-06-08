# Platform Configuration API

## Get Platform Configuration

```bash
GET /platform
```

```bash
curl -X GET "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Update Platform Configuration

```bash
PATCH /platform
```

```bash
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "supportedCurrencies": ["USD", "EUR", "BTC"],
    "webhookUrl": "https://your-domain.com/webhooks/grid",
    "name": "Your Platform Name"
  }'
```

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `supportedCurrencies` | string[] | List of currency codes |
| `webhookUrl` | string | Webhook endpoint URL |
| `name` | string | Platform name |
| `logoUrl` | string | Logo URL |
