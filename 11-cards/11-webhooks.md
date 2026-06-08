# Card Webhooks

Card webhook events and how to consume them.

## Card State Change

```json
{
  "eventType": "card.state_change",
  "data": {
    "cardId": "Card:...",
    "previousState": "PENDING_ISSUE",
    "newState": "ACTIVE"
  }
}
```

## Card Funding Source Change

```json
{
  "eventType": "card.funding_source_change",
  "data": {
    "cardId": "Card:...",
    "previousFundingSources": ["InternalAccount:old"],
    "fundingSources": ["InternalAccount:new"]
  }
}
```

## Handler Example

```javascript
app.post('/webhooks/grid', async (req, res) => {
  const { eventType, data } = req.body;

  if (eventType === 'card.state_change') {
    await db.cards.update(data.cardId, { status: data.newState });

    if (data.newState === 'ACTIVE') {
      await notifyUser(data.cardholderId, 'Your card is now active!');
    }
    if (data.newState === 'FROZEN') {
      await notifyUser(data.cardholderId, 'Your card has been frozen.');
    }
  }

  if (eventType === 'card.funding_source_change') {
    await db.cards.update(data.cardId, {
      fundingSources: data.fundingSources
    });
  }

  res.status(200).send('OK');
});
```
