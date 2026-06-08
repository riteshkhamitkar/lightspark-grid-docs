# Beneficial Owners

Add and manage beneficial owners, directors, and company officers for business customers.

## What is a Beneficial Owner?

A beneficial owner is an individual who:
- Owns 25% or more of the business (directly or indirectly)
- OR exercises significant control over the business

Grid requires verification of all beneficial owners as part of KYB.

## API Endpoints

### Create a Beneficial Owner

```bash
POST /beneficial-owners
```

```bash
curl --request POST \
  --url https://api.lightspark.com/grid/2025-10-13/beneficial-owners \
  --header 'Authorization: Basic <encoded-value>' \
  --header 'Content-Type: application/json' \
  --data '{
    "customerId": "Customer:019542f5-b3e7-1d02-0000-000000000001",
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
  }'
```

### Request Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `customerId` | string | Yes | Parent business customer ID |
| `personalInfo` | object | Yes | Personal information |
| `personalInfo.fullName` | string | Yes | Full legal name |
| `personalInfo.birthDate` | string | Yes | Date of birth (YYYY-MM-DD) |
| `personalInfo.nationality` | string | Yes | Country code |
| `personalInfo.address` | object | Yes | Residential address |
| `identification` | object | Yes | ID information |
| `identification.type` | string | Yes | `SSN`, `PASSPORT`, `ITIN`, etc. |
| `identification.number` | string | Yes | ID number |
| `ownershipPercentage` | number | Yes | Ownership percentage (0-100) |
| `role` | string | Yes | `BENEFICIAL_OWNER` or `CONTROL_PERSON` |

### Get a Beneficial Owner

```bash
GET /beneficial-owners/{id}
```

### Update a Beneficial Owner

```bash
PATCH /beneficial-owners/{id}
```

Only provided fields are updated:

```bash
curl --request PATCH \
  --url https://api.lightspark.com/grid/2025-10-13/beneficial-owners/BeneficialOwner:... \
  --header 'Authorization: Basic <encoded-value>' \
  --header 'Content-Type: application/json' \
  --data '{
    "ownershipPercentage": 35
  }'
```

### List Beneficial Owners

```bash
GET /beneficial-owners?customerId=Customer:...
```

## Roles

| Role | Description |
|------|-------------|
| `BENEFICIAL_OWNER` | Owns 25%+ of the business |
| `CONTROL_PERSON` | Significant control (CEO, CFO, etc.) |

## Ownership Requirements

- Must declare ALL individuals with 25%+ ownership
- Must declare at least one control person
- Total ownership can exceed 100% (if multiple owners)
- 0% ownership allowed for control persons

## Verification

Beneficial owners go through the same KYC process as individual customers:

1. Create beneficial owner record
2. Upload ID documents
3. Submit for verification
4. Status tracked via `GET /verifications`

## Sandbox Magic Values

Use magic suffixes in `lastName` to control KYC outcome:

| Suffix | Result |
|--------|--------|
| `001` | KYC PENDING |
| `002` | KYC REJECTED |
| Other | KYC APPROVED |

## Document Upload for Beneficial Owners

Upload ID documents using the documents API:

```bash
POST /documents
```

With:
- `documentHolderId`: The beneficial owner ID
- `documentHolderType`: `BENEFICIAL_OWNER`
- `documentType`: `GOVERNMENT_ID` or `PROOF_OF_ADDRESS`

## Best Practices

1. **Collect all owners** - Don't miss anyone with 25%+
2. **Verify control person** - Always include at least one
3. **Accurate percentages** - Be precise with ownership %
4. **Valid IDs** - Ensure ID documents are not expired
5. **Update promptly** - Update if ownership changes
