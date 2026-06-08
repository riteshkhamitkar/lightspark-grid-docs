# List Customers

Retrieve a list of customers with optional filtering parameters.

## API Endpoint

```bash
GET /customers
```

## Request

```bash
# List all customers
curl --request GET \
  --url https://api.lightspark.com/grid/2025-10-13/customers \
  --header 'Authorization: Basic <encoded-value>'

# Filter by KYC status
curl --request GET \
  --url 'https://api.lightspark.com/grid/2025-10-13/customers?kycStatus=APPROVED' \
  --header 'Authorization: Basic <encoded-value>'

# Filter by customer type
curl --request GET \
  --url 'https://api.lightspark.com/grid/2025-10-13/customers?customerType=INDIVIDUAL' \
  --header 'Authorization: Basic <encoded-value>'
```

## Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `platformCustomerId` | string | Filter by your platform ID |
| `customerType` | string | `INDIVIDUAL` or `BUSINESS` |
| `kycStatus` | string | Filter by KYC status |
| `region` | string | Filter by region/country |
| `email` | string | Filter by email |
| `page` | number | Page number (default: 1) |
| `limit` | number | Results per page (default: 20, max: 100) |

## Response

```json
{
  "data": [
    {
      "id": "Customer:019542f5-b3e7-1d02-0000-000000000001",
      "platformCustomerId": "ind-9f84e0c2",
      "customerType": "INDIVIDUAL",
      "kycStatus": "APPROVED",
      "fullName": "Jane Smith",
      "email": "jane@example.com",
      "createdAt": "2025-01-15T10:00:00Z"
    },
    {
      "id": "Customer:019542f5-b3e7-1d02-0000-000000000002",
      "platformCustomerId": "biz-abc123",
      "customerType": "BUSINESS",
      "kycStatus": "APPROVED",
      "fullName": "Acme Corporation",
      "email": "finance@acme.example.com",
      "createdAt": "2025-01-14T09:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 2,
    "hasMore": false
  }
}
```

## Pagination

Use `page` and `limit` parameters to paginate through results:

```bash
GET /customers?page=2&limit=50
```

Check `pagination.hasMore` to determine if more pages exist.
