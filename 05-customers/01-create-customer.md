# Create Customer

Register a new customer in the system.

## API Endpoint

```bash
POST /customers
```

## Request

### Individual Customer

```bash
curl --request POST \
  --url https://api.lightspark.com/grid/2025-10-13/customers \
  --header 'Authorization: Basic <encoded-value>' \
  --header 'Content-Type: application/json' \
  --data '
{
  "customerType": "INDIVIDUAL",
  "platformCustomerId": "ind-9f84e0c2",
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
  "identification": {
    "type": "SSN",
    "number": "123-45-6789"
  }
}'
```

### Business Customer

```bash
curl --request POST \
  --url https://api.lightspark.com/grid/2025-10-13/customers \
  --header 'Authorization: Basic <encoded-value>' \
  --header 'Content-Type: application/json' \
  --data '
{
  "customerType": "BUSINESS",
  "platformCustomerId": "biz-abc123",
  "region": "US",
  "currencies": ["USD"],
  "businessInfo": {
    "legalName": "Acme Corporation",
    "dbaName": "Acme",
    "registrationNumber": "DE123456789",
    "entityType": "CORPORATION",
    "yearEstablished": 2015,
    "website": "https://acme.example.com",
    "industry": "TECHNOLOGY",
    "description": "Software development company"
  },
  "address": {
    "line1": "456 Corporate Blvd",
    "city": "New York",
    "state": "NY",
    "postalCode": "10001",
    "country": "US"
  },
  "email": "finance@acme.example.com",
  "phone": "+1-555-987-6543",
  "controlPerson": {
    "fullName": "John CEO",
    "birthDate": "1980-05-10",
    "title": "Chief Executive Officer"
  }
}'
```

## Request Parameters

### Common Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `customerType` | string | Yes | `INDIVIDUAL` or `BUSINESS` |
| `platformCustomerId` | string | No | Your platform's customer identifier |
| `region` | string | Yes* | ISO country code for regulatory jurisdiction |
| `currencies` | string[] | No | Currency codes customer will use |
| `email` | string | No | Email address |
| `phone` | string | No | Phone number |
| `address` | object | No | Address object |
| `address.line1` | string | No | Street address line 1 |
| `address.line2` | string | No | Street address line 2 |
| `address.city` | string | No | City |
| `address.state` | string | No | State/Province |
| `address.postalCode` | string | No | Postal/ZIP code |
| `address.country` | string | No | ISO country code |

### Individual Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fullName` | string | Yes | Full legal name |
| `birthDate` | string | Yes | Date of birth (YYYY-MM-DD) |
| `nationality` | string | Yes | Country code of nationality |
| `identification` | object | No | ID information |
| `identification.type` | string | No | `SSN`, `PASSPORT`, `ITIN` |
| `identification.number` | string | No | ID number |

### Business Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `businessInfo` | object | Yes | Business information |
| `businessInfo.legalName` | string | Yes | Legal entity name |
| `businessInfo.dbaName` | string | No | Doing Business As name |
| `businessInfo.registrationNumber` | string | Yes | Business registration/Tax ID |
| `businessInfo.entityType` | string | Yes | `CORPORATION`, `LLC`, `PARTNERSHIP`, etc. |
| `businessInfo.yearEstablished` | number | No | Year company founded |
| `businessInfo.website` | string | No | Company website |
| `businessInfo.industry` | string | No | Industry code |
| `controlPerson` | object | No | Primary control person |

## Response

### 201 Created - Success

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
  "kycStatus": "PENDING",
  "createdAt": "2025-01-15T10:00:00Z",
  "updatedAt": "2025-01-15T10:00:00Z",
  "isDeleted": false
}
```

### 400 Bad Request

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Invalid request data",
  "details": [
    {
      "field": "birthDate",
      "message": "Invalid date format"
    }
  ]
}
```

### 409 Conflict

```json
{
  "error": "CONFLICT",
  "message": "Customer with this platformCustomerId already exists"
}
```

## Important Notes

1. **Auto-created accounts** - When you create a customer, internal accounts are automatically created for each currency in your platform configuration.
2. **KYC required** - Customers must complete KYC/KYB before sending/receiving payments.
3. **Unique platformCustomerId** - This ID must be unique across your platform.
4. **Immutable region** - The region field cannot be changed after creation.
5. **Sandbox behavior** - In sandbox, customers are automatically KYC-approved.

## Best Practices

1. **Use your own ID** - Set `platformCustomerId` to link with your internal records.
2. **Collect all required info** - Have full name, birth date, address ready.
3. **Start KYC immediately** - Generate KYC link right after creation.
4. **Validate input** - Check email format, phone format, date formats.
5. **Store the Grid ID** - Save the returned `Customer:...` ID in your database.

## Next Steps

After creating a customer:
1. [Generate KYC Link](../04-kyc-kyb-sumsub/03-hosted-kyc-link.md)
2. [Upload Documents](../04-kyc-kyb-sumsub/05-documents.md) (if needed)
3. [Submit for Verification](../04-kyc-kyb-sumsub/02-submit-customer-for-verification.md)
