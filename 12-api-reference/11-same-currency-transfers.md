# Same-Currency Transfers API

## Create a Transfer-Out Request

```bash
POST /transfer-outs
```

```bash
curl -X POST "$GRID_BASE_URL/transfer-outs" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "Customer:...",
    "sourceAccountId": "InternalAccount:...",
    "destinationAccountId": "ExternalAccount:...",
    "amount": "1000.00",
    "currency": "USD"
  }'
```

## Create a Transfer-In Request

```bash
POST /transfer-ins
```

```bash
curl -X POST "$GRID_BASE_URL/transfer-ins" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "Customer:...",
    "sourceAccountId": "ExternalAccount:...",
    "destinationAccountId": "InternalAccount:...",
    "amount": "1000.00",
    "currency": "USD"
  }'
```

Use for same-currency transfers where FX is not needed.
