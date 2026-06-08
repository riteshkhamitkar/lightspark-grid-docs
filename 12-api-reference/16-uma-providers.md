# Available UMA Providers API

## List Available Counterparty Providers

```bash
GET /uma/providers
```

```bash
curl -X GET "$GRID_BASE_URL/uma/providers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

### Response

```json
{
  "data": [
    {
      "umaAddress": "$provider@domain.com",
      "name": "Provider Name",
      "supportedCurrencies": ["USD", "EUR"]
    }
  ]
}
```

## Lookup UMA Address

```bash
GET /uma-addresses/{address}
```

```bash
curl -X GET "$GRID_BASE_URL/uma-addresses/\$user@domain.com" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

### Response

```json
{
  "umaAddress": "$user@domain.com",
  "supportedCurrencies": ["USD", "EUR"],
  "receivingLimits": {
    "daily": "10000.00",
    "monthly": "100000.00"
  }
}
```
