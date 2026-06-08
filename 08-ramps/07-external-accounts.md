# External Accounts for Ramps

Configure external bank accounts and crypto wallets for ramp destinations.

## Crypto Wallets

### Lightning Address

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "LIGHTNING_SPARK",
    "address": "$user@wallet.com",
    "currency": "BTC",
    "accountOwnerName": "User Name"
  }'
```

### UMA Address

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "UMA",
    "umaAddress": "$user@domain.com",
    "currency": "USD",
    "accountOwnerName": "User Name"
  }'
```

## Bank Accounts

Same as [Payouts external accounts](../07-payouts-and-b2b/07-external-accounts.md).

## Best Practices

1. **Validate addresses** - Check format before creating
2. **Test small first** - Send test amount
3. **Verify ownership** - Ensure user owns the wallet
