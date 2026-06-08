# Rewards Webhooks

Webhook events for rewards distribution.

## Key Events

### transaction.status_change

```json
{
  "eventType": "transaction.status_change",
  "data": {
    "transactionId": "Transaction:...",
    "status": "COMPLETED",
    "type": "TRANSFER_OUT",
    "amount": "0.001",
    "currency": "BTC",
    "sourceAmount": "50.00",
    "sourceCurrency": "USD"
  }
}
```

Handle by notifying user of completed reward.

### incoming_payment.received

For funding the platform account.

## Handler Example

```javascript
app.post('/webhooks/grid', async (req, res) => {
  const { eventType, data } = req.body;

  if (eventType === 'transaction.status_change') {
    if (data.type === 'TRANSFER_OUT' && data.currency === 'BTC') {
      // This is a reward
      await notifyRewardDelivered(data);
    }
  }

  res.status(200).send('OK');
});
```
