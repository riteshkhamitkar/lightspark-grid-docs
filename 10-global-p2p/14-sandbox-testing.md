# Global P2P Sandbox Testing

Test UMA payment flows safely without moving real funds.

## UMA Test Wallet

Grid provides a test UMA wallet for testing:
- `$test@grid-sandbox.com` - Always available for testing

## Test Flow

```bash
# 1. Create sender customer
# 2. Fund account
# 3. Create quote to test UMA address
curl -X POST "$GRID_BASE_URL/quotes" \
  -d '{
    "source": { "sourceType": "ACCOUNT", "accountId": "InternalAccount:usd-123" },
    "destination": { "destinationType": "UMA_ADDRESS", "umaAddress": "$test@grid-sandbox.com" },
    "amount": "10.00",
    "amountCurrency": "USD"
  }'
# 4. Execute -> SUCCESS (simulated)
```

## Testing Checklist

- [ ] UMA payment
- [ ] Bank account payment
- [ ] Receive UMA payment
- [ ] Quote expiration
- [ ] Invalid UMA address
- [ ] Insufficient funds
- [ ] Webhook events
