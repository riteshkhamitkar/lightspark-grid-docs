# Internal Accounts for Rewards

Learn how to manage and fund internal accounts for holding platform and customer funds.

Same as [Payouts internal accounts](../07-payouts-and-b2b/06-internal-accounts.md).

For rewards, you primarily need:
- **Platform USD account** - Holds funds for rewards
- May also need EUR, GBP, etc. depending on your funding currency

## Funding Your Rewards Account

### Bank Transfer

Get payment instructions:
```bash
curl -X GET "$GRID_BASE_URL/internal-accounts/InternalAccount:platform-usd" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Send ACH push or wire to the provided account details.

### Sandbox

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/.../fund" \
  -d '{"amount": "100000", "currency": "USD"}'
```

## Monitoring Balance

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts/InternalAccount:platform-usd" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Set up alerts for low balance to ensure uninterrupted rewards.
