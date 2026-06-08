# Get Customer by ID

Retrieve a customer by their system-generated ID.

## API Endpoint

```bash
GET /customers/{id}
```

## Request

```bash
curl --request GET \
  --url https://api.lightspark.com/grid/2025-10-13/customers/Customer:019542f5-b3e7-1d02-0000-000000000001 \
  --header 'Authorization: Basic <encoded-value>'
```

## Response

```json
{
  "id": "Customer:019542f5-b3e7-1d02-0000-000000000001",
  "platformCustomerId": "ind-9f84e0c2",
  "customerType": "INDIVIDUAL",
  "region": "US",
  "currencies": ["USD", "USDC"],
  "fullName": "Jane Smith",
  "birthDate": "1990-01-15",
  "nationality": "US",
  "email": "jane@example.com",
  "phone": "+1-555-123-4567",
  "address": {
    "line1": "123 Main Street",
    "line2": "Apt 4B",
    "city": "San Francisco",
    "state": "CA",
    "postalCode": "94105",
    "country": "US"
  },
  "kycStatus": "APPROVED",
  "createdAt": "2025-01-15T10:00:00Z",
  "updatedAt": "2025-01-15T12:00:00Z",
  "isDeleted": false
}
```

## Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Grid's unique customer ID |
| `platformCustomerId` | string | Your platform's identifier |
| `customerType` | string | `INDIVIDUAL` or `BUSINESS` |
| `kycStatus` | string | `NOT_STARTED`, `PENDING`, `IN_REVIEW`, `APPROVED`, `REJECTED` |
| `region` | string | ISO country code |
| `currencies` | string[] | Supported currencies |
| `fullName` | string | Customer's full name |
| `email` | string | Email address |
| `createdAt` | string | Creation timestamp |
| `updatedAt` | string | Last update timestamp |
| `isDeleted` | boolean | Whether customer is soft-deleted |

## 404 Not Found

```json
{
  "error": "NOT_FOUND",
  "message": "Customer not found"
}
```
