# Ramps Sandbox Testing

Test ramp flows safely without moving real funds.

## Magic Values

Same KYC magic values as other products.

## Testing On-Ramp

```bash
# 1. Create customer (KYC auto-approved)
# 2. Fund USD account
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/.../fund" \
  -d '{"amount": "1000", "currency": "USD"}'

# 3. Add BTC wallet external account
# 4. Create quote USD->BTC
# 5. Execute -> BTC "sent" (simulated)
```

## Testing Off-Ramp

```bash
# 1. Create customer
# 2. Fund BTC account
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/.../fund" \
  -d '{"amount": "0.5", "currency": "BTC"}'

# 3. Add bank account
# 4. Create quote BTC->USD
# 5. Execute -> USD "sent" (simulated)
```

## Testing Checklist

- [ ] On-ramp USD to BTC
- [ ] Off-ramp BTC to USD
- [ ] Quote expiration handling
- [ ] Insufficient funds error
- [ ] Invalid wallet address
- [ ] Webhook events
- [ ] Transaction status tracking
