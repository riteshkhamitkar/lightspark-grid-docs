# Internal Accounts for Payouts

Learn how to manage internal accounts for holding platform and customer funds.

## Overview

Internal accounts hold funds within Grid. For payouts:
- **Customer internal accounts** - Hold customer funds for their payouts
- **Platform internal accounts** - Hold your platform's operational funds

## Account Creation

Internal accounts are auto-created when a customer is created based on your platform's currency configuration.

## Listing Accounts

### Customer Accounts

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts?customerId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

### Platform Accounts

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts?accountOwnerType=PLATFORM" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Checking Balances

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts/InternalAccount:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Response:
```json
{
  "id": "InternalAccount:usd-123",
  "currency": "USD",
  "balance": "50000.00",
  "status": "ACTIVE",
  "paymentInstructions": {
    "accountNumber": "1234567890",
    "routingNumber": "021000021",
    "bankName": "Lightspark Bank"
  }
}
```

## Funding Accounts

### Bank Transfer (Production)

Use the `paymentInstructions` to send an ACH push or wire:
```bash
# Get instructions
curl -X GET "$GRID_BASE_URL/internal-accounts/InternalAccount:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"

# Send bank transfer using the provided account/routing numbers
```

### Sandbox Funding

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:.../fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "100000.00",
    "currency": "USD"
  }'
```

## Best Practices

1. **Monitor balances** - Ensure sufficient funds for payouts
2. **Auto-replenish** - Set up alerts for low balances
3. **Separate currencies** - Use separate accounts per currency
4. **Reconcile regularly** - Match balances with your ledger
5. **Secure funding instructions** - Don't expose instructions publicly
