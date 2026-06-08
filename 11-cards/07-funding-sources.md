# Funding Sources

Bind and update internal accounts as card funding sources.

## What is a Funding Source?

An internal account linked to a card. All card transactions are debited against this account.

## Binding Funding Source

When issuing a card:

```bash
curl -X POST "$GRID_BASE_URL/cards" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "cardholderId": "Customer:...",
    "fundingSources": ["InternalAccount:usd-123"]
  }'
```

## Multiple Funding Sources

A card can have multiple funding sources with priority:

```bash
curl -X POST "$GRID_BASE_URL/cards" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "cardholderId": "Customer:...",
    "fundingSources": ["InternalAccount:usd-primary", "InternalAccount:usd-backup"]
  }'
```

## Update Funding Sources

```bash
PATCH /cards/{id}
```

```bash
curl -X PATCH "$GRID_BASE_URL/cards/Card:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "fundingSources": ["InternalAccount:usd-new"]
  }'
```

## Webhook Event

```json
{
  "eventType": "card.funding_source_change",
  "data": {
    "cardId": "Card:...",
    "previousFundingSources": ["InternalAccount:old"],
    "fundingSources": ["InternalAccount:new"]
  }
}
```

## Best Practices

1. **Match currency** - Funding source currency = card currency
2. **Monitor balance** - Low balance = declined transactions
3. **Backup sources** - Multiple sources for reliability
4. **Auto-replenish** - Set up balance alerts
