# Card State Change Webhook

Webhook that is called when a card's lifecycle state changes.

## Event Type

```
card.state_change
```

## Payload

```json
{
  "eventType": "card.state_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "cardId": "Card:...",
    "previousState": "PENDING_ISSUE",
    "newState": "ACTIVE",
    "cardholderId": "Customer:..."
  }
}
```

## Handling

1. Update card status in your database
2. Notify cardholder
3. Enable/disable card features
4. Log state transition
