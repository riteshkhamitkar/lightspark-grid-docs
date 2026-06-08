# Paying Out Bitcoin Rewards

Send Bitcoin rewards to customers using the Grid API.

## Single Reward

```bash
# 1. Create quote
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": { "sourceType": "ACCOUNT", "accountId": "InternalAccount:platform-usd" },
    "destination": { "destinationType": "ACCOUNT", "accountId": "ExternalAccount:btc-456" },
    "amount": "10.00",
    "amountCurrency": "USD"
  }'

# 2. Execute
curl -X POST "$GRID_BASE_URL/quotes/Quote:.../execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Batch Rewards

```javascript
async function sendBatchRewards(recipients) {
  const results = [];

  for (const recipient of recipients) {
    try {
      const quote = await grid.quotes.create({
        source: { accountId: platformAccountId },
        destination: { accountId: recipient.walletId },
        amount: recipient.amountUSD,
        amountCurrency: 'USD'
      }, {
        idempotencyKey: `reward-${recipient.id}-${today}`
      });

      const tx = await grid.quotes.execute(quote.id);
      results.push({ recipient: recipient.id, status: 'success', tx: tx.id });
    } catch (error) {
      results.push({ recipient: recipient.id, status: 'failed', error: error.message });
    }
  }

  return results;
}
```

## Scheduled Rewards

For recurring rewards (e.g., monthly cashback):

```javascript
// Using a cron job or scheduler
cron.schedule('0 0 1 * *', async () => {
  const eligibleUsers = await calculateCashback();
  await sendBatchRewards(eligibleUsers);
});
```

## Reward Notifications

Notify users when rewards are sent:

```javascript
// After transaction completes
if (tx.status === 'COMPLETED') {
  await sendPushNotification(userId, {
    title: 'Reward Received!',
    body: `You received ${tx.destinationAmount} BTC ($${tx.sourceAmount} USD)`
  });
}
```

## Reward Display

Show users their reward history:

```bash
curl -X GET "$GRID_BASE_URL/transactions?customerId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Best Practices

1. **Use idempotency** - Prevent duplicate sends
2. **Log everything** - Track all reward attempts
3. **Handle failures** - Retry failed rewards
4. **Notify users** - Confirm reward delivery
5. **Show history** - Let users see past rewards
6. **Set limits** - Per-user reward caps
