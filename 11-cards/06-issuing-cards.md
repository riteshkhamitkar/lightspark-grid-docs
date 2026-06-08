# Issuing Cards

Create a virtual card and observe its lifecycle.

## Issue Card

```bash
POST /cards
```

```bash
curl -X POST "$GRID_BASE_URL/cards" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "cardholderId": "Customer:cardholder-001",
    "fundingSources": ["InternalAccount:usd-123"],
    "cardName": "My Virtual Card",
    "spendingLimits": {
      "daily": "1000.00",
      "monthly": "10000.00"
    }
  }'
```

## Response

```json
{
  "id": "Card:card-abc123",
  "cardholderId": "Customer:cardholder-001",
  "fundingSources": ["InternalAccount:usd-123"],
  "lastFour": "4242",
  "status": "PENDING_ISSUE",
  "createdAt": "2025-01-15T10:00:00Z"
}
```

## Card Lifecycle

```
PENDING_KYC -> PENDING_ISSUE -> ACTIVE
                                    |
                                FROZEN
                                    |
                                CLOSED
```

## Get Card

```bash
curl -X GET "$GRID_BASE_URL/cards/Card:card-abc123" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## List Cards

```bash
curl -X GET "$GRID_BASE_URL/cards?cardholderId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Best Practices

1. **One card per funding source** - Or multiple if needed
2. **Set spending limits** - Prevent overspending
3. **Name cards** - Help users identify cards
4. **Track status** - Monitor lifecycle changes
