# Update Customer by ID

Update a customer's metadata by their system-generated ID.

## API Endpoint

```bash
PATCH /customers/{id}
```

## Request

```bash
curl --request PATCH \
  --url https://api.lightspark.com/grid/2025-10-13/customers/Customer:019542f5-b3e7-1d02-0000-000000000001 \
  --header 'Authorization: Basic <encoded-value>' \
  --header 'Content-Type: application/json' \
  --data '{
    "email": "newemail@example.com",
    "phone": "+1-555-999-8888",
    "address": {
      "line1": "789 New Address",
      "city": "Los Angeles",
      "state": "CA",
      "postalCode": "90001",
      "country": "US"
    }
  }'
```

## Updateable Fields

| Field | Type | Notes |
|-------|------|-------|
| `email` | string | Must be valid email format |
| `phone` | string | Must be valid phone format |
| `address` | object | Full address object |
| `fullName` | string | For individuals |

## Immutable Fields

These fields cannot be changed:
- `customerType`
- `region`
- `birthDate`
- `nationality`
- `createdAt`

## Response

Returns the updated customer object (same format as GET).

## 404 Not Found

```json
{
  "error": "NOT_FOUND",
  "message": "Customer not found"
}
```
