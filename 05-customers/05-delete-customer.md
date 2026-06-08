# Delete Customer by ID

Delete a customer by their system-generated ID.

## API Endpoint

```bash
DELETE /customers/{id}
```

## Request

```bash
curl --request DELETE \
  --url https://api.lightspark.com/grid/2025-10-13/customers/Customer:019542f5-b3e7-1d02-0000-000000000001 \
  --header 'Authorization: Basic <encoded-value>'
```

## Important Notes

1. **Soft delete** - Customers are soft-deleted, not permanently removed.
2. **No payments** - Customer must have no pending payments.
3. **Cascade** - Associated accounts may also be deleted.
4. **Irreversible** - Cannot be undone through the API.

## Response

### 204 No Content
Customer successfully deleted.

### 400 Bad Request
```json
{
  "error": "VALIDATION_ERROR",
  "message": "Customer has pending transactions"
}
```

### 404 Not Found
```json
{
  "error": "NOT_FOUND",
  "message": "Customer not found"
}
```

## Before Deleting

Ensure:
- No pending transactions
- No active quotes
- Balance is zero or withdrawn
- No active cards

Consider updating the customer status instead of deleting if historical records are needed.
