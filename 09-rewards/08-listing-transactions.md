# Listing Transactions

Query and track Bitcoin reward payment history with filtering and pagination.

## API Endpoint

```bash
GET /transactions
```

## Filter by Type

```bash
# All reward transactions
curl -X GET "$GRID_BASE_URL/transactions?type=TRANSFER_OUT" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"

# For specific time period
curl -X GET "$GRID_BASE_URL/transactions?fromDate=2025-01-01&toDate=2025-01-31" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"

# For specific customer
curl -X GET "$GRID_BASE_URL/transactions?customerId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Reward Reporting

```javascript
async function generateRewardReport(startDate, endDate) {
  const transactions = await grid.transactions.list({
    type: 'TRANSFER_OUT',
    fromDate: startDate,
    toDate: endDate,
    currency: 'BTC'
  });

  const summary = {
    totalRewards: transactions.length,
    totalBTC: transactions.reduce((sum, tx) => sum + parseFloat(tx.amount), 0),
    totalUSD: transactions.reduce((sum, tx) => sum + parseFloat(tx.sourceAmount), 0),
    successful: transactions.filter(tx => tx.status === 'COMPLETED').length,
    failed: transactions.filter(tx => tx.status === 'FAILED').length
  };

  return summary;
}
```

## Export to CSV

```javascript
async function exportRewardsToCSV(startDate, endDate) {
  const transactions = await grid.transactions.list({ fromDate: startDate, toDate: endDate });

  const csv = [
    ['Date', 'Recipient', 'Amount BTC', 'Amount USD', 'Status'],
    ...transactions.map(tx => [
      tx.createdAt,
      tx.customerName,
      tx.destinationAmount,
      tx.sourceAmount,
      tx.status
    ])
  ].map(row => row.join(',')).join('\n');

  return csv;
}
```
