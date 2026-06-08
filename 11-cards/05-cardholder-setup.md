# Cardholder Setup

Prepare a customer to receive a card.

## Requirements

1. Customer created
2. KYC status: `APPROVED`
3. Internal account with sufficient balance

## Check KYC Status

```bash
curl -X GET "$GRID_BASE_URL/customers/Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Response:
```json
{
  "kycStatus": "APPROVED"
}
```

If not approved, complete KYC first.

## Set Up Funding Source

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts?customerId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Ensure account has sufficient balance for expected card usage.

## Funding Source Requirements

- Currency must match card's currency
- Status must be ACTIVE
- Sufficient balance for authorizations

## Best Practices

1. **Verify KYC** - Never issue without APPROVED KYC
2. **Fund appropriately** - Ensure sufficient balance
3. **Set limits** - Consider per-transaction limits
4. **Monitor usage** - Track spending patterns
