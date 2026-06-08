# Internal Accounts for Global P2P

Create and manage internal accounts for P2P payments.

Same as [Payouts internal accounts](../07-payouts-and-b2b/06-internal-accounts.md).

## Multi-Currency Accounts

Customers may have accounts in multiple currencies:

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts?customerId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Response may include USD, EUR, BTC accounts.

## UMA-Linked Accounts

UMA payments automatically route to the correct internal account based on currency.
