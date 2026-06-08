# Payouts Webhooks

Webhook events relevant to Payouts & B2B operations.

## Key Webhook Events

### transaction.status_change

Payment status changed:

```json
{
  "eventType": "transaction.status_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "transactionId": "Transaction:tx-001",
    "status": "COMPLETED",
    "previousStatus": "IN_FLIGHT",
    "type": "TRANSFER_OUT",
    "amount": "500.00",
    "currency": "USD",
    "sourceAccountId": "InternalAccount:usd-123",
    "destinationAccountId": "ExternalAccount:spei-456",
    "quoteId": "Quote:quote-789",
    "customerId": "Customer:cust-001",
    "completedAt": "2025-01-15T10:30:00Z"
  }
}
```

### incoming_payment.received

Funds received into internal account:

```json
{
  "eventType": "incoming_payment.received",
  "timestamp": "2025-01-15T09:00:00Z",
  "data": {
    "accountId": "InternalAccount:usd-123",
    "customerId": "Customer:cust-001",
    "amount": "10000.00",
    "currency": "USD",
    "reference": "ACH-PUSH-001"
  }
}
```

### customer.status_change

Customer KYC status changed:

```json
{
  "eventType": "customer.status_change",
  "timestamp": "2025-01-15T08:00:00Z",
  "data": {
    "customerId": "Customer:cust-001",
    "previousStatus": { "kycStatus": "PENDING" },
    "newStatus": { "kycStatus": "APPROVED" }
  }
}
```

### internal_account.status_change

Account balance updated:

```json
{
  "eventType": "internal_account.status_change",
  "timestamp": "2025-01-15T09:00:00Z",
  "data": {
    "accountId": "InternalAccount:usd-123",
    "balance": "10500.00",
    "previousBalance": "500.00",
    "currency": "USD"
  }
}
```

## Handler Implementation

```javascript
app.post('/webhooks/grid', async (req, res) => {
  // Verify signature
  if (!verifyWebhook(req)) {
    return res.status(401).send('Invalid');
  }

  const { eventType, data } = req.body;

  switch (eventType) {
    case 'transaction.status_change':
      await handleTransactionUpdate(data);
      break;
    case 'incoming_payment.received':
      await handleDeposit(data);
      break;
    case 'customer.status_change':
      await handleCustomerUpdate(data);
      break;
    case 'internal_account.status_change':
      await handleBalanceUpdate(data);
      break;
  }

  res.status(200).send('OK');
});

async function handleTransactionUpdate(data) {
  await db.payments.update(data.quoteId, {
    status: data.status,
    completedAt: data.completedAt
  });

  // Notify customer
  if (data.status === 'COMPLETED') {
    await notifyCustomer(data.customerId, 'Payment sent successfully!');
  } else if (data.status === 'FAILED') {
    await notifyCustomer(data.customerId, 'Payment failed. Contact support.');
  }
}

async function handleDeposit(data) {
  await db.accounts.updateBalance(data.accountId, data.balance);
  await notifyCustomer(data.customerId, `Received $${data.amount}`);
}
```

## Webhook Security

Always verify signatures:

```javascript
function verifyWebhook(req) {
  const signature = req.headers['x-grid-signature'];
  const timestamp = req.headers['x-grid-timestamp'];

  const expected = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(`${timestamp}.${JSON.stringify(req.body)}`)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expected, 'hex')
  );
}
```

## Best Practices

1. **Process async** - Return 200 immediately
2. **Verify signatures** - Always authenticate webhooks
3. **Handle all events** - Every event type needs handler
4. **Log events** - Keep audit trail
5. **Retry logic** - Handle transient failures
6. **Monitor delivery** - Alert on failures
