# Sending Payments

Sending payments to UMA addresses and bank accounts.

## UMA Payment

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:usd-123"
    },
    "destination": {
      "destinationType": "UMA_ADDRESS",
      "umaAddress": "$receiver@domain.com"
    },
    "amount": "50.00",
    "amountCurrency": "USD"
  }'
```

## Bank Account Payment

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:usd-123"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "ExternalAccount:bank-456"
    },
    "amount": "100.00",
    "amountCurrency": "USD"
  }'
```

## UMA Lookup

Before sending, you can look up a UMA address:

```bash
curl -X GET "$GRID_BASE_URL/uma-addresses/\$receiver@domain.com" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Response shows supported currencies and limits.

## Best Practices

1. **Lookup first** - Verify UMA address exists
2. **Show fees** - Display total cost before sending
3. **Confirm details** - Have user confirm recipient
4. **Track status** - Show progress after sending
