# Payouts Quickstart

Send your first cross-border payment.

## Prerequisites

- Sandbox API credentials
- cURL or API client
- Webhook endpoint (optional, for notifications)

## Step 1: Set Environment Variables

```bash
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
export GRID_CLIENT_ID="YOUR_SANDBOX_CLIENT_ID"
export GRID_CLIENT_SECRET="YOUR_SANDBOX_CLIENT_SECRET"
```

## Step 2: Onboard a Customer

### Create the Customer

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "cust-001",
    "region": "US",
    "fullName": "Jane Doe",
    "birthDate": "1990-01-15",
    "nationality": "US",
    "email": "jane@example.com",
    "address": {
      "line1": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postalCode": "94105",
      "country": "US"
    }
  }'
```

### Generate KYC Link

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../kyc-link" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

In sandbox, KYC is auto-approved. In production, redirect the customer to the KYC URL.

### Get Internal Account

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts?customerId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Step 3: Fund the Internal Account

### Sandbox: Use Fund Endpoint

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:.../fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "10000.00",
    "currency": "USD"
  }'
```

### Production: Deposit via Bank Transfer

Get payment instructions from the internal account:
```bash
curl -X GET "$GRID_BASE_URL/internal-accounts/InternalAccount:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Use the `paymentInstructions` to send a bank transfer.

## Step 4: Add a Beneficiary (External Account)

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

## Step 5: Create a Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:usd-123"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "ExternalAccount:spei-456"
    },
    "amount": "500.00",
    "amountCurrency": "USD"
  }'
```

Response:
```json
{
  "id": "Quote:quote-789",
  "sourceAmount": "500.00",
  "sourceAmountCurrency": "USD",
  "destinationAmount": "8500.00",
  "destinationAmountCurrency": "MXN",
  "exchangeRate": "17.00",
  "fees": "7.50",
  "feesCurrency": "USD",
  "expiresAt": "2025-01-15T10:05:00Z"
}
```

## Step 6: Execute the Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:quote-789/execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json"
```

## Step 7: Monitor the Transaction

### Via Webhook

Listen for `transaction.status_change` events.

### Via API

```bash
curl -X GET "$GRID_BASE_URL/transactions?quoteId=Quote:quote-789" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Complete Flow Summary

```
1. Create Customer -> Customer:...
2. Complete KYC -> kycStatus: APPROVED
3. Get Internal Account -> InternalAccount:...
4. Fund Account -> balance: 10000.00 USD
5. Add External Account -> ExternalAccount:...
6. Create Quote -> Quote:... (locked rate)
7. Execute Quote -> Transaction:...
8. Monitor via webhooks -> COMPLETED
```

## Next Steps

- [Platform Configuration](./04-platform-configuration.md)
- [Sending Payments](./09-sending-payments.md)
- [Reconciliation](./11-reconciliation.md)
