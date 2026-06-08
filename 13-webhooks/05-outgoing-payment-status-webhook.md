# Outgoing Payment Status Webhook

Webhook that is called when an outgoing payment's status changes.

## Event Type

```
transaction.status_change
```

## Payload

```json
{
  "eventType": "transaction.status_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "transactionId": "Transaction:...",
    "status": "COMPLETED",
    "previousStatus": "IN_FLIGHT",
    "type": "TRANSFER_OUT",
    "amount": "500.00",
    "currency": "USD",
    "sourceAccountId": "InternalAccount:...",
    "destinationAccountId": "ExternalAccount:...",
    "quoteId": "Quote:...",
    "customerId": "Customer:..."
  }
}
```

## Handling

1. Update payment status in your database
2. Notify customer of completion/failure
3. Trigger post-payment logic
4. Update reporting metrics
