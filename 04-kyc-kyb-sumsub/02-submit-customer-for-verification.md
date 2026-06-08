# Submit Customer for Verification

Trigger KYC (individual) or KYB (business) verification for a customer.

## API Endpoint

```bash
POST /verifications
```

## Request

```bash
curl --request POST \
  --url https://api.lightspark.com/grid/2025-10-13/verifications \
  --header 'Authorization: Basic <encoded-value>' \
  --header 'Content-Type: application/json' \
  --data '
{
  "customerId": "Customer:019542f5-b3e7-1d02-0000-000000000001"
}'
```

## Request Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `customerId` | string | Yes | Customer ID to verify |

## Response

### Success with Missing Info

```json
{
  "id": "Verification:019542f5-b3e7-1d02-0000-000000000001",
  "customerId": "Customer:019542f5-b3e7-1d02-0000-000000000001",
  "verificationStatus": "RESOLVE_ERRORS",
  "errors": [
    {
      "resourceId": "Customer:019542f5-b3e7-1d02-0000-000000000001",
      "type": "MISSING_FIELD",
      "field": "customer.address.line1",
      "reason": "Business address line 1 is required"
    },
    {
      "resourceId": "Customer:019542f5-b3e7-1d02-0000-000000000001",
      "type": "MISSING_PROOF_OF_ADDRESS_DOCUMENT",
      "acceptedDocumentTypes": ["PROOF_OF_ADDRESS"],
      "reason": "Proof of address document is required"
    },
    {
      "resourceId": "BeneficialOwner:019542f5-b3e7-1d02-0000-000000000002",
      "type": "MISSING_FIELD",
      "field": "personalInfo.birthDate",
      "reason": "Date of birth is required for beneficial owners"
    }
  ],
  "createdAt": "2025-10-03T12:00:00Z"
}
```

### Success - Approved

```json
{
  "id": "Verification:019542f5-b3e7-1d02-0000-000000000001",
  "customerId": "Customer:019542f5-b3e7-1d02-0000-000000000001",
  "verificationStatus": "APPROVED",
  "kycStatus": "APPROVED",
  "errors": [],
  "createdAt": "2025-10-03T12:00:00Z",
  "completedAt": "2025-10-03T12:05:00Z"
}
```

## Error Types

### MISSING_FIELD
A required field is missing. The `field` property tells you which field.

### MISSING_DOCUMENT
A required document is missing. The `acceptedDocumentTypes` tells you what to upload.

### MISSING_PROOF_OF_ADDRESS_DOCUMENT
Proof of address document is required.

### INVALID_FIELD
A field value is invalid.

## What to Collect for KYB

Before submitting a BUSINESS customer, collect the following via `POST /customers`, `POST /beneficial-owners`, and `POST /documents`:

### Business Identifying Information
- Entity full legal name
- Doing Business As (DBA) name, if applicable
- Physical address - principal place of business
- Countries of operation
- Identification number - U.S. taxpayer identification number, or, for a foreign business without one, alternative government-issued documentation certifying the existence of the business

### Ownership and Control Structure
Collected for:
- **One control person** - an individual with significant responsibility to control, manage, or direct the legal entity
- **All beneficial owners** - every individual who owns 25% or more, directly or indirectly

For each person provide:
- Full name
- Date of birth
- Address
- Identification number:
  - U.S. persons: SSN or ITIN
  - Non-U.S. persons: one or more of: ITIN, passport (with country of issuance), alien identification card, or another government-issued photo ID evidencing nationality or residence

### Required Documents
- Company formation and existence documents (certificate of incorporation, articles of association, etc.)
- Proof of ownership and control structure (organization and ownership chart, shareholder agreements, operating agreements, register of members, or certification of controlling person and beneficial owners)
- Proof of address dated within the last 3 months (utility bill, bank statement, lease agreement, or official correspondence)
- Tax ID or equivalent identifying-number documents
- For non-U.S. beneficial owners: passport plus one additional government-issued ID

## Re-submitting After Errors

Call the endpoint again after resolving errors:

```bash
POST /verifications
```

With the same `customerId`. Grid will re-check all requirements.

## Get Verification by ID

```bash
GET /verifications/{id}
```

```bash
curl --request GET \
  --url https://api.lightspark.com/grid/2025-10-13/verifications/Verification:... \
  --header 'Authorization: Basic <encoded-value>'
```

## List Verifications

```bash
GET /verifications?customerId=Customer:...&status=APPROVED
```

| Query Parameter | Description |
|----------------|-------------|
| `customerId` | Filter by customer |
| `status` | Filter by status |

## Webhook Events

`verification.status_change` - Sent when verification status changes:

```json
{
  "eventType": "verification.status_change",
  "data": {
    "verificationId": "Verification:...",
    "customerId": "Customer:...",
    "status": "APPROVED",
    "previousStatus": "PENDING"
  }
}
```

## Best Practices

1. **Submit early** - Start verification as soon as customer is created
2. **Handle errors** - Show customers exactly what's missing
3. **Re-submit promptly** - Don't wait after fixing errors
4. **Poll or webhook** - Check status via webhooks, not polling
5. **Store verification ID** - Keep the verification ID for reference
