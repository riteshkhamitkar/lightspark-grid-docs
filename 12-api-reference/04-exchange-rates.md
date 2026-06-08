# Exchange Rates API

## Get Exchange Rates

```bash
GET /exchange-rates
```

```bash
curl -X GET "$GRID_BASE_URL/exchange-rates" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| `fromCurrency` | Source currency |
| `toCurrency` | Destination currency |

### Response

```json
{
  "fromCurrency": "USD",
  "toCurrency": "MXN",
  "rate": "17.05",
  "timestamp": "2025-01-15T10:00:00Z",
  "expiresAt": "2025-01-15T10:05:00Z"
}
```

## Notes

- Rates are cached for ~5 minutes
- Rates include platform-specific fees
- Use quotes for locked rates
