# External Accounts for Payouts

Add and manage external bank accounts, wallets, and payment destinations for withdrawals and payouts.

## Overview

External accounts are the destinations for your payouts. They represent:
- Vendor bank accounts
- Contractor wallets
- Supplier payment details
- Beneficiary accounts

## Supported Account Types

| Type | Identifier | Countries |
|------|-----------|-----------|
| ACH | Account + Routing Number | US |
| WIRE | Account + Wire Routing | US |
| SEPA | IBAN | EU |
| SEPA_INSTANT | IBAN | EU |
| SPEI | CLABE | MX |
| PIX | PIX Key | BR |
| UPI | UPI ID | IN |
| INSTAPAY | Account Number | PH |
| PESONET | Account Number | PH |
| BANK_TRANSFER | Account Details | Various |

## Creating an External Account

```bash
POST /customers/{customerId}/external-accounts
```

### US ACH Example

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "ACH",
    "accountNumber": "1234567890",
    "routingNumber": "021000021",
    "currency": "USD",
    "accountOwnerName": "Jane Doe"
  }'
```

### Mexico SPEI Example

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "SPEI",
    "clabe": "002115016003269411",
    "currency": "MXN",
    "accountOwnerName": "Maria Garcia"
  }'
```

### Brazil PIX Example

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "PIX",
    "pixKey": "maria@email.com",
    "pixKeyType": "EMAIL",
    "currency": "BRL",
    "accountOwnerName": "Maria Silva"
  }'
```

### India UPI Example

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "UPI",
    "upiId": "user@upi",
    "currency": "INR",
    "accountOwnerName": "Raj Kumar"
  }'
```

### Philippines InstaPay Example

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "INSTAPAY",
    "accountNumber": "1234567890",
    "bankName": "BDO Unibank",
    "currency": "PHP",
    "accountOwnerName": "Juan Dela Cruz"
  }'
```

### SEPA Example

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "SEPA",
    "iban": "DE89370400440532013000",
    "currency": "EUR",
    "accountOwnerName": "Hans Mueller"
  }'
```

## Looking Up Banks

For some rails, you can look up supported institutions:

```bash
curl -X GET "$GRID_BASE_URL/discoveries?country=PH&currency=PHP" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Response:
```json
{
  "data": [
    { "bankName": "BDO Unibank", "displayName": "BDO Unibank", "country": "PH", "currency": "PHP" },
    { "bankName": "BPI", "displayName": "Bank of the Philippine Islands", "country": "PH", "currency": "PHP" }
  ]
}
```

## External Account Status

| Status | Description |
|--------|-------------|
| `PENDING` | Created, awaiting verification |
| `ACTIVE` | Verified and ready for use |
| `VERIFIED` | Additional verification completed |
| `FAILED` | Verification failed |

## Best Practices

1. **Verify account details** - Double-check account numbers
2. **Name matching** - Ensure account owner name matches
3. **Test first** - Send small test amount
4. **Store securely** - Encrypt sensitive account data
5. **Keep updated** - Handle account changes
