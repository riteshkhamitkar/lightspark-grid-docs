# Sandbox Testing

Use Grid sandbox magic values to trigger specific API responses.

The Grid sandbox environment simulates real payment flows without moving real money. You can control test outcomes using special account number patterns and test addresses.

## KYC/KYB Verification Magic Values

### Individual Customer KYC

The **last 3 characters** of the `lastName` in `personalInfo` determine the KYC status outcome:

| Suffix | kycStatus | Behavior |
|--------|-----------|----------|
| `001` | `PENDING` | KYC verification remains pending |
| `002` | `REJECTED` | KYC verification is rejected |
| Any other | `APPROVED` | KYC verification is approved |

**Example:**
```json
{
  "fullName": "Jane Doe",
  "lastName": "TestUser001",
  ...
}
```

### Business Customer KYB

The **last 3 digits** of the `registrationNumber` in `businessInfo` determine the KYB status outcome:

| Suffix | kybStatus | Behavior |
|--------|-----------|----------|
| `001` | `PENDING` | KYB verification remains pending |
| `002` | `REJECTED` | KYB verification is rejected |
| Any other | `APPROVED` | KYB verification is approved |

**Example:**
```json
{
  "businessInfo": {
    "registrationNumber": "DE123456001"
  }
}
```

### Beneficial Owner KYC

The **last 3 characters** of the `lastName` in `personalInfo` determine the KYC status:

| Suffix | kycStatus | Behavior |
|--------|-----------|----------|
| `001` | `PENDING` | KYC remains pending |
| `002` | `REJECTED` | KYC is rejected |
| Any other | `APPROVED` | KYC is approved |

## External Account Magic Values

The flows for creating external accounts in sandbox are the same as in production. The **last 3 digits** of an external account's primary identifier determine the test scenario.

### Account Number Suffix Patterns

For identifiers with a domain part (e.g., PIX email keys), append test digits to the username portion: `testuser.002@pix.com.br`

### Beneficiary Name Verification

For account types that support beneficiary name verification, use account identifiers with a `1xx` suffix to trigger verification scenarios:

| Suffix | Result |
|--------|--------|
| `100` | Name verified successfully |
| `101` | Name verification failed |

## Transfer Testing

### Transfer In (External to Internal)

1. Create external account (customer's bank)
2. Create internal account (Grid)
3. Create transfer-in request
4. Use sandbox simulate endpoint to fund

### Sandbox Fund Endpoint

```bash
POST /sandbox/internal-accounts/{id}/fund
```

Simulate receiving funds into an internal account:
```bash
curl -X POST "https://api.lightspark.com/grid/2025-10-13/sandbox/internal-accounts/InternalAccount:.../fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "1000.00",
    "currency": "USD"
  }'
```

### Transfer Out (Internal to External)

Create a quote and execute:
```bash
POST /quotes
POST /quotes/{id}/execute
```

## Quote Testing

### Creating Quotes

Same as production. The quote response includes realistic (but simulated) rates and fees.

### Executing Quotes

Execution happens immediately in sandbox (no real settlement delay).

## UMA Payment Testing

### Simulate Incoming UMA Payments

```bash
POST /sandbox/uma/payments/simulate
```

Simulate receiving a UMA payment:
```bash
curl -X POST "https://api.lightspark.com/grid/2025-10-13/sandbox/uma/payments/simulate" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "100.00",
    "currency": "USD",
    "receiverUmaAddress": "$user@your-domain.com"
  }'
```

## Global Account Magic Values

### Email OTP Code

In sandbox, email OTP codes are always: `000000`

### Passkey WebAuthn Ceremony

In sandbox, WebAuthn ceremonies are automatically approved (no real hardware needed).

### OAuth (OIDC) Token

Sandbox accepts any valid JWT format token for OAuth testing.

### Wallet Signature Header

For testing Global Account withdrawals, any valid base64 string is accepted as a signature in sandbox.

## Card Testing

### Simulate Card Authorization

```bash
POST /sandbox/cards/{card_id}/authorize
```

Simulate an authorization on a card:
```bash
curl -X POST "https://api.lightspark.com/grid/2025-10-13/sandbox/cards/Card:.../authorize" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "50.00",
    "currency": "USD",
    "merchantName": "Test Merchant"
  }'
```

### Simulate Card Clearing

```bash
POST /sandbox/cards/{card_id}/clear
```

### Simulate Card Return (Refund)

```bash
POST /sandbox/cards/{card_id}/return
```

## Webhook Testing

### Send Test Webhook

```bash
POST /sandbox/webhooks/send-test
```

Trigger a test webhook to your configured endpoint:
```bash
curl -X POST "https://api.lightspark.com/grid/2025-10-13/sandbox/webhooks/send-test" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "customer.status_change"
  }'
```

## Complete Test Scenarios

### Scenario 1: Successful Payout
```
1. Create customer (lastName: "User123") -> KYC APPROVED
2. Fund internal account via /sandbox/fund
3. Create external account (suffix: 000 -> ACTIVE)
4. Create quote
5. Execute quote -> SUCCESS
```

### Scenario 2: KYC Rejection
```
1. Create customer (lastName: "User002") -> KYC REJECTED
2. Try to create quote -> Error: KYC_NOT_APPROVED
```

### Scenario 3: Insufficient Funds
```
1. Create customer -> KYC APPROVED
2. Don't fund internal account (or fund with small amount)
3. Create quote for large amount
4. Execute -> Error: INSUFFICIENT_FUNDS
```

### Scenario 4: Global Account Withdrawal
```
1. Create customer with USDB currency
2. Fund Global Account via /sandbox/fund
3. Add external bank account
4. Create withdrawal quote
5. Execute with any base64 signature -> SUCCESS
```

## Best Practices for Sandbox Testing

1. **Test all scenarios** - Success, failure, edge cases
2. **Use magic values consistently** - Document which values you use
3. **Automate tests** - Include sandbox tests in CI/CD
4. **Test webhooks** - Verify your webhook handling
5. **Test idempotency** - Verify safe retry behavior
6. **Load test** - Test with realistic volumes
7. **Clean up** - Delete test data periodically
