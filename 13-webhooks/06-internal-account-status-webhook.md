# Internal Account Status Webhook

Webhook that is called when the status of an internal account changes, including balance updates.

## Event Type

```
internal_account.status_change
```

## Payload

```json
{
  "eventType": "internal_account.status_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "accountId": "InternalAccount:...",
    "customerId": "Customer:...",
    "balance": "5000.00",
    "previousBalance": "4500.00",
    "status": "ACTIVE",
    "currency": "USD"
  }
}
```

## Handling

1. Update displayed balance
2. Check for low balance alerts
3. Update internal ledger
4. Notify customer if significant change
