# Reconciliation

Match Grid transactions with your internal systems.

## Why Reconcile?

- Ensure accuracy between Grid and your ledger
- Catch discrepancies early
- Support financial reporting
- Facilitate audit compliance

## Matching Fields

Use these fields to match Grid transactions with your records:

| Grid Field | Your System | Notes |
|-----------|-------------|-------|
| `id` | Grid transaction ID | Store in your DB |
| `platformCustomerId` | Your customer ID | Link customer |
| `quoteId` | Your payment reference | Link to quote |
| `amount` | Payment amount | Match exactly |
| `currency` | Currency code | Verify match |
| `status` | Payment status | Compare states |
| `createdAt` | Initiation time | Match timeframe |

## Reconciliation Process

### Daily Reconciliation

1. **Export Grid transactions**
   ```bash
   curl -X GET "$GRID_BASE_URL/transactions?fromDate=2025-01-15&toDate=2025-01-15" \
     -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
   ```

2. **Export your payment records**
   - Query your database for same date range

3. **Match transactions**
   - Match by Grid transaction ID
   - Verify amounts match
   - Compare status fields

4. **Identify discrepancies**
   - Grid transactions not in your system
   - Your records not in Grid
   - Status mismatches
   - Amount mismatches

5. **Resolve discrepancies**
   - Investigate unmatched items
   - Update records as needed
   - Escalate if needed

### Automated Reconciliation

```python
# Pseudocode
def reconcile(grid_transactions, internal_records):
    matched = []
    unmatched_grid = []
    unmatched_internal = []

    # Build lookup by transaction ID
    grid_by_id = {tx.id: tx for tx in grid_transactions}
    internal_by_id = {r.grid_tx_id: r for r in internal_records}

    # Match transactions
    for tx_id, grid_tx in grid_by_id.items():
        if tx_id in internal_by_id:
            internal = internal_by_id[tx_id]
            if amounts_match(grid_tx, internal) and statuses_match(grid_tx, internal):
                matched.append((grid_tx, internal))
            else:
                unmatched_grid.append(grid_tx)
        else:
            unmatched_grid.append(grid_tx)

    # Find internal records not in Grid
    for tx_id, internal in internal_by_id.items():
        if tx_id not in grid_by_id:
            unmatched_internal.append(internal)

    return matched, unmatched_grid, unmatched_internal
```

## Webhook-Based Reconciliation

Instead of polling, use webhooks:

```javascript
app.post('/webhooks/grid', async (req, res) => {
  const { eventType, data } = req.body;

  if (eventType === 'transaction.status_change') {
    // Update your internal record
    await db.payments.updateByGridTxId(data.id, {
      status: data.status,
      updatedAt: data.updatedAt
    });

    // Trigger reconciliation check
    await reconcileSingle(data.id);
  }

  res.status(200).send('OK');
});
```

## Handling Edge Cases

### Pending Transactions
- Grid: `IN_FLIGHT`
- Your system: `PROCESSING`
- Action: Wait for completion webhook

### Failed Transactions
- Grid: `FAILED`
- Your system: `PROCESSING`
- Action: Update to `FAILED`, notify customer, allow retry

### Duplicate Transactions
- Use idempotency keys to prevent
- Check for duplicate Grid TX IDs

### Timing Differences
- Grid timestamp vs. your timestamp
- Allow for timezone differences
- Use createdAt for matching

## Reporting

### Daily Summary Report

| Metric | Value |
|--------|-------|
| Total Transactions | 150 |
| Successful | 148 |
| Failed | 2 |
| Total Volume | $75,000 |
| Average Amount | $500 |
| Matched | 150/150 |

### Monthly Reconciliation Report

Prepare for finance team:
- Opening balance
- Total inflows
- Total outflows
- Closing balance
- Unmatched items
- Discrepancy resolution

## Best Practices

1. **Reconcile daily** - Catch issues early
2. **Automate matching** - Script the process
3. **Investigate unmatched** - Every item needs explanation
4. **Document procedures** - Clear reconciliation process
5. **Audit trail** - Log all reconciliation actions
6. **Escalation path** - Who to contact for issues
