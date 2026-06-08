# KYC/KYB Verification Overview

Grid provides comprehensive identity verification through hosted KYC/KYB flows powered by SumSub.

## What is KYC/KYB?

**KYC (Know Your Customer)** - Identity verification for individuals
**KYB (Know Your Business)** - Verification for business entities including beneficial owners

## Verification Flow

### For Individuals (KYC)

1. **Create Customer** - Create the customer record via API
2. **Generate Hosted KYC Link** - Create a SumSub verification URL
3. **Customer Completes Verification** - Customer submits documents via hosted flow
4. **Track Status** - Poll or receive webhooks for status updates
5. **Approval/Rejection** - Customer approved or rejected

### For Businesses (KYB)

1. **Create Business Customer** - Create with business information
2. **Add Beneficial Owners** - Submit all 25%+ owners and control persons
3. **Upload Documents** - Company formation docs, proof of address, IDs
4. **Submit for Verification** - Call verification endpoint
5. **Resolve Errors** - Fix any missing information
6. **Track Status** - Monitor verification progress

## Verification Statuses

| Status | Description |
|--------|-------------|
| `PENDING` | Verification submitted, awaiting review |
| `IN_REVIEW` | Under review by verification team |
| `APPROVED` | Verification successful |
| `REJECTED` | Verification failed |
| `RESOLVE_ERRORS` | Missing information, needs correction |

## Customer KYC Status

| Status | Meaning |
|--------|---------|
| `NOT_STARTED` | No verification initiated |
| `PENDING` | Verification in progress |
| `IN_REVIEW` | Under manual review |
| `APPROVED` | Verified and approved |
| `REJECTED` | Verification rejected |

## Hosted KYC Flow

Grid uses SumSub for identity verification. You can:

1. **Use Grid's Hosted Flow** (Recommended) - Generate a link, redirect customer
2. **Bring Your Own KYC** - Use your existing KYC provider (contact Lightspark)

## Required Information

### Individual KYC

- Full legal name
- Date of birth
- Nationality
- Address (line1, city, postal code, country)
- Government-issued ID document
- Selfie (in some cases)

### Business KYB

**Business Information:**
- Full legal entity name
- DBA (Doing Business As) name
- Physical address
- Countries of operation
- Tax ID / Registration number
- Company formation documents

**Beneficial Owners (25%+ ownership):**
- Full name
- Date of birth
- Address
- Government-issued ID
- Ownership percentage

**Control Person:**
- Full name
- Date of birth
- Address
- Government-issued ID
- Role in company

**Required Documents:**
- Certificate of incorporation
- Articles of association
- Proof of address (utility bill, bank statement, < 3 months)
- Tax ID documents
- Organization chart showing ownership
- ID documents for all beneficial owners

## Regional Requirements

| Region | Special Requirements |
|--------|---------------------|
| US | SSN or ITIN |
| EU | National ID or Passport |
| UK | Proof of address required |
| Other | Passport + additional ID |

## Integration Steps

### Step 1: Create Customer
```bash
POST /customers
```

### Step 2: Generate KYC Link
```bash
POST /customers/{id}/kyc-link
```

### Step 3: Redirect Customer
Redirect customer to the hosted URL:
```
https://verify.sumsub.com/... (returned from API)
```

### Step 4: Handle Webhooks
Listen for `verification.status_change` webhooks.

### Step 5: Check Status
```bash
GET /verifications/{id}
```

## Timing

- **Automated checks:** Instant to minutes
- **Manual review:** Hours to 1-2 business days
- **Complex cases:** Up to 5 business days

## Compliance Notes

- All customers must be verified before sending/receiving
- Ongoing monitoring may trigger re-verification
- Sanctions screening is performed automatically
- PEP (Politically Exposed Person) checks included
- Adverse media screening available

## Sandbox Testing

Use magic values to test KYC/KYB flows without real verification:

| Magic Suffix | Result |
|-------------|--------|
| `001` | Status stays PENDING |
| `002` | Status becomes REJECTED |
| Any other | Status becomes APPROVED |

See [Sandbox Testing](../03-getting-started/03-sandbox-testing.md) for details.
