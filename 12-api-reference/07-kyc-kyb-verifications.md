# KYC/KYB Verifications API

## Submit Customer for Verification

```bash
POST /verifications
```

## Get a Verification

```bash
GET /verifications/{id}
```

## List Verifications

```bash
GET /verifications
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| `customerId` | Filter by customer |
| `status` | Filter by status |

## Create a Beneficial Owner

```bash
POST /beneficial-owners
```

## Get a Beneficial Owner

```bash
GET /beneficial-owners/{id}
```

## Update a Beneficial Owner

```bash
PATCH /beneficial-owners/{id}
```

## List Beneficial Owners

```bash
GET /beneficial-owners?customerId=Customer:...
```

## Beneficial Owner Request Body

```json
{
  "customerId": "Customer:...",
  "personalInfo": {
    "fullName": "John Smith",
    "birthDate": "1985-03-20",
    "nationality": "US",
    "address": {
      "line1": "456 Oak Street",
      "city": "Los Angeles",
      "state": "CA",
      "postalCode": "90001",
      "country": "US"
    }
  },
  "identification": {
    "type": "SSN",
    "number": "123-45-6789"
  },
  "ownershipPercentage": 30,
  "role": "BENEFICIAL_OWNER"
}
```
