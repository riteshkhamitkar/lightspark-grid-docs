# Rewards Sandbox Testing

Test your rewards integration in the Grid sandbox environment.

## Test Flow

```bash
# 1. Fund platform account
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/.../fund" \
  -d '{"amount": "10000", "currency": "USD"}'

# 2. Create recipient
curl -X POST "$GRID_BASE_URL/customers" \
  -d '{"customerType": "INDIVIDUAL", "fullName": "Test User", ...}'

# 3. Add BTC wallet
curl -X POST "$GRID_BASE_URL/customers/.../external-accounts" \
  -d '{"accountType": "LIGHTNING_SPARK", "address": "$test@wallet.com"}'

# 4. Create and execute quote -> SUCCESS
```

## Testing Checklist

- [ ] Fund platform account
- [ ] Create recipient
- [ ] Add wallet
- [ ] Create quote USD->BTC
- [ ] Execute quote
- [ ] Verify webhook received
- [ ] Check transaction status
- [ ] List transactions
- [ ] Test idempotency
- [ ] Test error cases (funds, invalid address)
