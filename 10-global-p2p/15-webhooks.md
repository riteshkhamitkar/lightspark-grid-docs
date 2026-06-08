# Global P2P Webhooks

Security best practices for webhook verification.

## Key Events

### incoming_payment.received

```json
{
  "eventType": "incoming_payment.received",
  "data": {
    "customerId": "Customer:...",
    "accountId": "InternalAccount:...",
    "amount": "100.00",
    "currency": "USD",
    "sourceUmaAddress": "$sender@domain.com"
  }
}
```

### transaction.status_change

```json
{
  "eventType": "transaction.status_change",
  "data": {
    "transactionId": "Transaction:...",
    "status": "COMPLETED",
    "type": "TRANSFER_OUT",
    "umaAddress": "$receiver@domain.com"
  }
}
```

## Handler Example

```javascript
app.post('/webhooks/grid', async (req, res) => {
  const { eventType, data } = req.body;

  if (eventType === 'incoming_payment.received') {
    // Credit user's account
    await creditUser(data.customerId, data.amount, data.currency);
    await notifyUser(data.customerId, `Received $${data.amount} from ${data.sourceUmaAddress}`);
  }

  res.status(200).send('OK');
});
```
