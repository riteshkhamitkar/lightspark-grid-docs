# Discoveries API

## List Available Receiving Institution Names

```bash
GET /discoveries
```

```bash
curl -X GET "$GRID_BASE_URL/discoveries?country=PH&currency=PHP" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| `country` | ISO 3166-1 alpha-2 country code |
| `currency` | ISO 4217 currency code |

### Response

```json
{
  "data": [
    {
      "bankName": "BDO Unibank",
      "displayName": "BDO Unibank",
      "country": "PH",
      "currency": "PHP"
    }
  ]
}
```

## Use Case

Look up supported banks for a specific corridor before creating external accounts.
