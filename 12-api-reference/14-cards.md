# Cards API

## Issue a Card

```bash
POST /cards
```

```bash
curl -X POST "$GRID_BASE_URL/cards" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "cardholderId": "Customer:...",
    "fundingSources": ["InternalAccount:..."],
    "cardName": "My Card",
    "spendingLimits": {
      "daily": "1000.00",
      "monthly": "10000.00"
    }
  }'
```

## Get a Card

```bash
GET /cards/{id}
```

## List Cards

```bash
GET /cards
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| `cardholderId` | Filter by cardholder |
| `status` | Filter by status |

## Update a Card

```bash
PATCH /cards/{id}
```

```bash
curl -X PATCH "$GRID_BASE_URL/cards/Card:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "FROZEN",
    "fundingSources": ["InternalAccount:new"]
  }'
```

## Card States

| State | Description |
|-------|-------------|
| `PENDING_KYC` | Awaiting KYC |
| `PENDING_ISSUE` | Being provisioned |
| `ACTIVE` | Ready for use |
| `FROZEN` | Temporarily disabled |
| `CLOSED` | Permanently closed |
