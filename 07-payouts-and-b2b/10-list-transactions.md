# List Transactions

Query and filter payment history with powerful filtering and pagination options.

## API Endpoint

```bash
GET /transactions
```

## Request

```bash
# All transactions
curl -X GET "$GRID_BASE_URL/transactions" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"

# Filter by customer
curl -X GET "$GRID_BASE_URL/transactions?customerId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"

# Filter by status
curl -X GET "$GRID_BASE_URL/transactions?status=COMPLETED" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"

# Filter by date range
curl -X GET "$GRID_BASE_URL/transactions?fromDate=2025-01-01&toDate=2025-01-31" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"

# Filter by currency
curl -X GET "$GRID_BASE_URL/transactions?currency=USD" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `customerId` | string | Filter by customer |
| `platformCustomerId` | string | Filter by your customer ID |
| `status` | string | `PENDING`, `IN_FLIGHT`, `COMPLETED`, `FAILED` |
| `currency` | string | Filter by currency |
| `fromDate` | string | Start date (ISO 8601) |
| `toDate` | string | End date (ISO 8601) |
| `type` | string | `TRANSFER_IN`, `TRANSFER_OUT`, `CONVERSION` |
| `page` | number | Page number |
| `limit` | number | Results per page (max 100) |

## Response

```json
{
  "data": [
    {
      "id": "Transaction:tx-001",
      "type": "TRANSFER_OUT",
      "status": "COMPLETED",
      "amount": "500.00",
      "currency": "USD",
      "sourceAccountId": "InternalAccount:usd-123",
      "destinationAccountId": "ExternalAccount:spei-456",
      "customerId": "Customer:cust-001",
      "quoteId": "Quote:quote-789",
      "createdAt": "2025-01-15T10:00:00Z",
      "completedAt": "2025-01-15T10:02:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "hasMore": true
  }
}
```

## Get Single Transaction

```bash
curl -X GET "$GRID_BASE_URL/transactions/Transaction:tx-001" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Pagination

```bash
# Page 2
curl -X GET "$GRID_BASE_URL/transactions?page=2&limit=50" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Best Practices

1. **Use filters** - Don't fetch all transactions
2. **Paginate** - Handle large result sets
3. **Cache results** - Cache for short periods
4. **Index by customer** - Fast lookup per customer
5. **Export capability** - Allow CSV export
