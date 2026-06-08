# Global Account Webhooks

Receive webhook events for funding, withdrawals, and other account state changes on Global Accounts.

## Relevant Webhook Events

### transaction.status_change

Sent when a transaction involving a Global Account changes status.

```json
{
  "eventType": "transaction.status_change",
  "data": {
    "transactionId": "Transaction:...",
    "status": "COMPLETED",
    "previousStatus": "IN_FLIGHT",
    "sourceAccountId": "InternalAccount:ga-...",
    "destinationAccountId": "ExternalAccount:...",
    "amount": "100.00",
    "currency": "USDB"
  }
}
```

### internal_account.status_change

Sent when Global Account balance or status changes.

```json
{
  "eventType": "internal_account.status_change",
  "data": {
    "accountId": "InternalAccount:ga-...",
    "customerId": "Customer:...",
    "balance": "900.00",
    "previousBalance": "1000.00",
    "status": "ACTIVE",
    "currency": "USDB"
  }
}
```

### customer.status_change

KYC status changes affect Global Account capabilities.

```json
{
  "eventType": "customer.status_change",
  "data": {
    "customerId": "Customer:...",
    "newStatus": {
      "kycStatus": "APPROVED"
    }
  }
}
```

### incoming_payment.received

When funds are received into a Global Account.

```json
{
  "eventType": "incoming_payment.received",
  "data": {
    "customerId": "Customer:...",
    "accountId": "InternalAccount:ga-...",
    "amount": "500.00",
    "currency": "USDB",
    "source": "ExternalAccount:..."
  }
}
```

## Handling Global Account Webhooks

### Example Handler

```javascript
app.post('/webhooks/grid', async (req, res) => {
  const { eventType, data } = req.body;

  switch (eventType) {
    case 'transaction.status_change': {
      if (isGlobalAccount(data.sourceAccountId)) {
        await handleGlobalAccountWithdrawal(data);
      }
      if (isGlobalAccount(data.destinationAccountId)) {
        await handleGlobalAccountDeposit(data);
      }
      break;
    }

    case 'internal_account.status_change': {
      if (isGlobalAccount(data.accountId)) {
        await updateGlobalAccountBalance(data);
      }
      break;
    }

    case 'incoming_payment.received': {
      if (isGlobalAccount(data.accountId)) {
        await notifyCustomerOfDeposit(data);
      }
      break;
    }
  }

  res.status(200).send('OK');
});

function isGlobalAccount(accountId) {
  // Check if account is an EMBEDDED_WALLET
  return accountId.startsWith('InternalAccount:ga-');
}
```

## Webhook Security

Always verify webhook signatures before processing:

```javascript
function verifyWebhook(payload, signature, timestamp, secret) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expected, 'hex')
  );
}
```

## Best Practices

1. **Filter by account type** - Check if account is EMBEDDED_WALLET
2. **Update balances** - Keep balance display current
3. **Notify customers** - Push notifications for deposits/withdrawals
4. **Track transactions** - Store transaction IDs for reference
5. **Handle failures** - Alert customers of failed withdrawals
