# Incoming Payment Webhook

Webhook that is called when an incoming payment is received by a customer's UMA address or internal account.

## Event Type

```
incoming_payment.received
```

## Payload

```json
{
  "eventType": "incoming_payment.received",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "customerId": "Customer:...",
    "accountId": "InternalAccount:...",
    "amount": "100.00",
    "currency": "USD",
    "sourceUmaAddress": "$sender@domain.com",
    "transactionId": "Transaction:..."
  }
}
```

## Handling

1. Credit the customer's balance
2. Notify customer of received payment
3. Update transaction history
4. Trigger any business logic (e.g., release held funds)
