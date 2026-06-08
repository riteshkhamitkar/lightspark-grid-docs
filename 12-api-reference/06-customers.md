# Customers API

## Add a New Customer

```bash
POST /customers
```

## Get Customer by ID

```bash
GET /customers/{id}
```

## Update Customer by ID

```bash
PATCH /customers/{id}
```

## Delete Customer by ID

```bash
DELETE /customers/{id}
```

## List Customers

```bash
GET /customers
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| `platformCustomerId` | Filter by your ID |
| `customerType` | `INDIVIDUAL` or `BUSINESS` |
| `kycStatus` | Filter by KYC status |
| `region` | Filter by region |
| `page` | Page number |
| `limit` | Results per page |

## Upload Customers via CSV

```bash
POST /customers/bulk-upload
```

## Get Bulk Import Job Status

```bash
GET /customers/bulk-upload/{jobId}
```

## Generate Hosted KYC Link

```bash
POST /customers/{customerId}/kyc-link
```

### Response

```json
{
  "url": "https://verify.sumsub.com/...",
  "token": "eyJhbGci...",
  "expiresAt": "2025-01-15T11:00:00Z"
}
```
