# KYC/KYB Sandbox Magic Values

Magic values for testing KYC/KYB flows in sandbox.

## Overview

In sandbox, you can trigger specific KYC/KYB verification outcomes using magic suffixes in customer and beneficial owner fields. These let you test different verification flows without waiting for real review.

## Individual Customer KYC

### Magic Suffix

The **last 3 characters** of the `lastName` field determine the KYC outcome:

| Suffix | kycStatus | Behavior |
|--------|-----------|----------|
| `001` | `PENDING` | KYC verification remains pending |
| `002` | `REJECTED` | KYC verification is rejected |
| Any other | `APPROVED` | KYC verification is approved |

### Examples

**Approved customer (default):**
```json
{
  "customerType": "INDIVIDUAL",
  "fullName": "Jane Doe",
  "lastName": "Doe",
  ...
}
// Result: kycStatus = APPROVED
```

**Pending customer:**
```json
{
  "customerType": "INDIVIDUAL",
  "fullName": "Jane Pending",
  "lastName": "Pending001",
  ...
}
// Result: kycStatus = PENDING
```

**Rejected customer:**
```json
{
  "customerType": "INDIVIDUAL",
  "fullName": "Jane Rejected",
  "lastName": "Rejected002",
  ...
}
// Result: kycStatus = REJECTED
```

## Business Customer KYB

### Magic Suffix

The **last 3 digits** of the `registrationNumber` in `businessInfo` determine the KYB outcome:

| Suffix | kybStatus | Behavior |
|--------|-----------|----------|
| `001` | `PENDING` | KYB verification remains pending |
| `002` | `REJECTED` | KYB verification is rejected |
| Any other | `APPROVED` | KYB verification is approved |

### Examples

**Approved business:**
```json
{
  "customerType": "BUSINESS",
  "businessInfo": {
    "legalName": "Acme Corp",
    "registrationNumber": "DE123456789"
  },
  ...
}
// Result: kybStatus = APPROVED
```

**Pending business:**
```json
{
  "customerType": "BUSINESS",
  "businessInfo": {
    "legalName": "Acme Corp",
    "registrationNumber": "DE123456001"
  },
  ...
}
// Result: kybStatus = PENDING
```

**Rejected business:**
```json
{
  "customerType": "BUSINESS",
  "businessInfo": {
    "legalName": "Acme Corp",
    "registrationNumber": "DE123456002"
  },
  ...
}
// Result: kybStatus = REJECTED
```

## Beneficial Owner KYC

### Magic Suffix

The **last 3 characters** of the `lastName` in `personalInfo` determine the individual KYC status:

| Suffix | kycStatus | Behavior |
|--------|-----------|----------|
| `001` | `PENDING` | KYC verification remains pending |
| `002` | `REJECTED` | KYC verification is rejected |
| Any other | `APPROVED` | KYC verification is approved |

### Example

```json
{
  "personalInfo": {
    "fullName": "John Smith",
    "lastName": "Smith002"
  },
  ...
}
// Result: KYC REJECTED
```

## Testing Scenarios

### Scenario 1: Successful Onboarding
```
1. Create individual customer (lastName: "Doe")
2. KYC auto-approved
3. Create business customer (registrationNumber: "DE123")
4. Add beneficial owners (lastName: "Smith")
5. Submit KYB - all approved
```

### Scenario 2: KYC Pending
```
1. Create customer (lastName: "Test001")
2. KYC stays PENDING
3. Poll for status updates
4. Test your pending state UI
```

### Scenario 3: KYC Rejected
```
1. Create customer (lastName: "Test002")
2. KYC auto-rejected
3. Test rejection handling
4. Verify account restrictions
```

### Scenario 4: Mixed KYB Results
```
1. Create business (registrationNumber: "DE123") -> APPROVED
2. Add owner 1 (lastName: "Smith") -> APPROVED
3. Add owner 2 (lastName: "Jones001") -> PENDING
4. Overall KYB stays PENDING until all owners approved
```

## Important Notes

1. **Magic values only work in sandbox**
2. **Suffix must be exactly 3 characters** at the end
3. **Case sensitive** - use exactly as documented
4. **Multiple conditions** - each field is evaluated independently
5. **Real verification** - production uses actual SumSub verification

## Quick Reference

| What | Field | Suffix | Result |
|------|-------|--------|--------|
| Individual KYC | `lastName` | `001` | PENDING |
| Individual KYC | `lastName` | `002` | REJECTED |
| Business KYB | `registrationNumber` | `001` | PENDING |
| Business KYB | `registrationNumber` | `002` | REJECTED |
| Beneficial Owner | `lastName` | `001` | PENDING |
| Beneficial Owner | `lastName` | `002` | REJECTED |
