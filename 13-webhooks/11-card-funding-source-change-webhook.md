# Card Funding Source Change Webhook

Webhook that is called when the funding sources bound to a card change.

## Event Type

```
card.funding_source_change
```

## Payload

```json
{
  "eventType": "card.funding_source_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "cardId": "Card:...",
    "previousFundingSources": ["InternalAccount:old"],
    "fundingSources": ["InternalAccount:new"]
  }
}
```

## Handling

1. Update funding source mapping
2. Verify new source has sufficient balance
3. Notify cardholder of change
4. Update card details display
