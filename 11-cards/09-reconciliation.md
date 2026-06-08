# Card Reconciliation

How card transactions reconcile, and what exceptions to act on.

## Transaction Types

| Type | Description |
|------|-------------|
| `AUTHORIZATION` | Hold placed on funding source |
| `CLEARING` | Actual debit after authorization |
| `REFUND` | Return of funds |
| `REVERSAL` | Cancellation of authorization |

## Reconciliation Flow

```
Authorization -> (time passes) -> Clearing
     |                                    |
     -> Reversal (if cancelled)     -> Refund (if returned)
```

### Authorization

When card is used:
1. Real-time check against funding source balance
2. Amount is held (reserved)
3. `CARD.AUTHORIZATION` webhook sent

### Clearing

After merchant settles:
1. Held amount is actually debited
2. May differ from authorization (tips, adjustments)
3. `transaction.status_change` webhook sent

### Refund

If customer returns purchase:
1. Merchant initiates refund
2. Funds credited back to funding source
3. `CARD.REFUND` webhook sent

## Webhook Events

### Card State Change

```json
{
  "eventType": "card.state_change",
  "data": {
    "cardId": "Card:...",
    "previousState": "ACTIVE",
    "newState": "FROZEN"
  }
}
```

### Card Funding Source Change

```json
{
  "eventType": "card.funding_source_change",
  "data": {
    "cardId": "Card:...",
    "fundingSources": ["InternalAccount:new-source"]
  }
}
```

## Exception Handling

| Exception | Action |
|-----------|--------|
| Authorization without clearing | Release hold after timeout |
| Clearing exceeds authorization | Debit actual amount |
| Partial refund | Credit refund amount |
| Dispute | Freeze card, investigate |

## Best Practices

1. **Match authorizations to clearings** - Track both
2. **Handle timing differences** - Authorizations hold, clearings deduct
3. **Monitor exceptions** - Alert on unmatched items
4. **Reconcile daily** - Match with funding source transactions
