# Payouts Sandbox Testing

Test your payouts integration in the Grid sandbox environment.

## Magic Values

### KYC/KYB

| Suffix | Result |
|--------|--------|
| `001` | PENDING |
| `002` | REJECTED |
| Other | APPROVED |

### External Accounts

| Account Number Suffix | Result |
|----------------------|--------|
| `000` | ACTIVE, transfers succeed |
| `001` | PENDING verification |
| `002` | FAILED verification |
| `003` | Transfer fails |
| `1xx` | Name verification scenarios |

## Test Scenarios

### Scenario 1: Successful Payout Flow

```bash
# 1. Create customer -> KYC APPROVED
curl -X POST "$GRID_BASE_URL/customers" ...

# 2. Fund account
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/.../fund" ...

# 3. Add external account (suffix: 000 -> ACTIVE)
curl -X POST "$GRID_BASE_URL/customers/.../external-accounts" \
  -d '{"accountNumber": "123456000", ...}'

# 4. Create quote
curl -X POST "$GRID_BASE_URL/quotes" ...

# 5. Execute -> SUCCESS
curl -X POST "$GRID_BASE_URL/quotes/.../execute" ...
```

### Scenario 2: Insufficient Funds

```bash
# Don't fund account (or fund with small amount)
# Create quote for large amount
# Execute -> Error: INSUFFICIENT_FUNDS
```

### Scenario 3: KYC Rejected Customer

```bash
# Create customer with lastName ending in 002
curl -X POST "$GRID_BASE_URL/customers" \
  -d '{"lastName": "Test002", ...}'
# KYC REJECTED
# Try to create quote -> Error: KYC_NOT_APPROVED
```

### Scenario 4: Quote Expiration

```bash
# Create quote
# Wait for expiration (5 minutes in sandbox)
# Try to execute -> Error: QUOTE_EXPIRED
```

### Scenario 5: Bank Rejection

```bash
# Add external account with suffix 003
curl -X POST "$GRID_BASE_URL/customers/.../external-accounts" \
  -d '{"accountNumber": "123456003", ...}'

# Execute quote -> Transaction FAILED with BANK_REJECTED
```

## Testing Webhooks

```bash
# Send test webhook
curl -X POST "$GRID_BASE_URL/sandbox/webhooks/send-test" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"eventType": "transaction.status_change"}'
```

## Automated Testing

```javascript
// Example Jest test
describe('Payout Flow', () => {
  test('successful USD to MXN payout', async () => {
    // 1. Create customer
    const customer = await grid.customers.create({
      customerType: 'INDIVIDUAL',
      fullName: 'Test User',
      ...
    });
    expect(customer.kycStatus).toBe('APPROVED');

    // 2. Fund account
    await grid.sandbox.fundAccount(customer.accounts[0].id, '10000', 'USD');

    // 3. Add external account
    const extAccount = await grid.externalAccounts.create(customer.id, {
      accountType: 'SPEI',
      clabe: '002115016000',
      ...
    });

    // 4. Create and execute quote
    const quote = await grid.quotes.create({
      source: { accountId: customer.accounts[0].id },
      destination: { accountId: extAccount.id },
      amount: '500',
      amountCurrency: 'USD'
    });

    const tx = await grid.quotes.execute(quote.id);
    expect(tx.status).toBe('COMPLETED');
  });
});
```

## Testing Checklist

- [ ] Customer creation with KYC
- [ ] Account funding
- [ ] External account creation
- [ ] Quote creation and execution
- [ ] Cross-currency payout
- [ ] Same-currency transfer
- [ ] UMA payment
- [ ] Error scenarios (funds, expired quote, KYC)
- [ ] Webhook handling
- [ ] Transaction listing
- [ ] Reconciliation
