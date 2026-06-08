# Internal Accounts for Ramps

Manage internal accounts for holding fiat and crypto balances for ramp operations.

## Account Types

| Currency | Type | Use |
|----------|------|-----|
| USD | CHECKING | Fiat balance |
| BTC | CHECKING | Crypto balance |
| USDC | CHECKING | Stablecoin balance |

## Funding Fiat Accounts

Same as [Payouts funding](../07-payouts-and-b2b/08-depositing-funds.md).

## Checking Crypto Balances

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts/InternalAccount:btc-123" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Response:
```json
{
  "id": "InternalAccount:btc-123",
  "currency": "BTC",
  "balance": "0.5",
  "status": "ACTIVE"
}
```

## Best Practices

1. **Separate accounts** - Different accounts per currency
2. **Monitor balances** - Alert on low balances
3. **Reconcile** - Match with your ledger
