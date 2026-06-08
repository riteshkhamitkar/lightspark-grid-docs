# Freezing & Closing Cards

Freeze, unfreeze, and close cards via the signed-retry pattern.

## Freeze Card

Temporarily disable a card:

```bash
curl -X PATCH "$GRID_BASE_URL/cards/Card:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "FROZEN"
  }'
```

When frozen:
- New authorizations are declined
- In-flight settlements continue
- Can be unfrozen

## Unfreeze Card

Re-enable a frozen card:

```bash
curl -X PATCH "$GRID_BASE_URL/cards/Card:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "ACTIVE"
  }'
```

## Close Card

Permanently close a card:

```bash
curl -X PATCH "$GRID_BASE_URL/cards/Card:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "CLOSED"
  }'
```

When closed:
- Cannot be reopened
- No new transactions
- Existing transactions settle

## Webhook Events

```json
{
  "eventType": "card.state_change",
  "data": {
    "cardId": "Card:...",
    "previousState": "ACTIVE",
    "newState": "FROZEN"
  }
}
```

## When to Freeze

- Suspected fraud
- User request
- Lost device
- Travel hold

## When to Close

- Card expired
- User request (permanent)
- Fraud confirmed
- Account closure

## Best Practices

1. **Allow self-freeze** - Let users freeze their own cards
2. **Quick response** - Freeze immediately on suspicion
3. **Clear communication** - Tell user why card was frozen
4. **Easy unfreeze** - Simple process to reactivate
5. **Irreversible close** - Warn before closing
