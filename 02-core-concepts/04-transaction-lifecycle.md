# Transaction Lifecycle

Follow a payment from creation to settlement.

## Transaction States

### Status Flow

```
PENDING -> IN_FLIGHT -> COMPLETED
    |          |
    -> FAILED  -> CANCELLED
```

| Status | Description |
|--------|-------------|
| `PENDING` | Transaction created, awaiting processing |
| `IN_FLIGHT` | Payment is being processed |
| `COMPLETED` | Payment completed successfully |
| `FAILED` | Payment failed (see error details) |
| `CANCELLED` | Payment was cancelled |

## Transaction Types

| Type | Description |
|------|-------------|
| `TRANSFER_IN` | Funds entering Grid (deposit) |
| `TRANSFER_OUT` | Funds leaving Grid (withdrawal/payout) |
| `CONVERSION` | Currency conversion transaction |
| `CARD_AUTHORIZATION` | Card authorization hold |
| `CARD_CLEARING` | Card transaction settlement |
| `CARD_REFUND` | Card refund |

## Payment Flow Stages

### 1. Quote Creation
- Customer/platform decides to make a payment
- Quote is created with locked rate and fees
- Quote has expiration time (1-5 minutes typically)

### 2. Quote Execution
- Platform calls execute quote endpoint
- Transaction is created with status `PENDING`
- Funds are reserved from source account

### 3. Processing
- Transaction moves to `IN_FLIGHT`
- Grid routes payment through optimal rails
- For cross-currency: FX conversion happens
- For crypto: on-chain transaction submitted

### 4. Settlement
- Local rail payment sent (ACH, SEPA, PIX, etc.)
- Or crypto transaction confirmed on-chain
- Transaction moves to `COMPLETED`

### 5. Notification
- Webhook sent with final status
- Platform updates internal records
- Customer notified (if applicable)

## Timing Estimates

| Rail Type | Typical Settlement Time |
|-----------|------------------------|
| Lightning Network | Seconds |
| UMA | Seconds |
| SEPA Instant | Seconds |
| PIX | Seconds |
| SEPA | Same day / 1 business day |
| ACH | 1-3 business days |
| Wire | Same day |
| UPI | Seconds - minutes |
| CLABE | Same day |

## Reconciliation

### Matching Transactions

Use these fields to match Grid transactions with your internal records:
- `platformCustomerId` - Your customer reference
- `quoteId` - Links to the original quote
- `transaction.id` - Grid's unique transaction ID
- `reference` - Your custom reference (if provided)

### Webhook Events for Reconciliation

- `transaction.created` - New transaction created
- `transaction.status_updated` - Status changed
- `transaction.completed` - Transaction completed
- `transaction.failed` - Transaction failed

### Best Practices

1. **Store transaction IDs** - Keep Grid transaction IDs in your database
2. **Listen for webhooks** - Don't poll; use webhooks for status updates
3. **Handle failures gracefully** - Have retry logic and customer communication
4. **Reconcile daily** - Match Grid transactions with your ledger
5. **Handle edge cases** - Partial failures, reversals, chargebacks

## Error Handling

### Common Transaction Failures

| Error Code | Description | Action |
|-----------|-------------|--------|
| `INSUFFICIENT_FUNDS` | Source account has insufficient balance | Notify customer, retry after funding |
| `INVALID_ACCOUNT` | Destination account invalid | Verify account details |
| `BANK_REJECTED` | Receiving bank rejected the payment | Contact beneficiary, verify details |
| `EXPIRED` | Quote expired before execution | Create new quote |
| `LIMIT_EXCEEDED` | Amount exceeds limits | Request limit increase or split payment |
| `COMPLIANCE_HOLD` | Payment held for compliance review | Wait for review completion |

### Retrying Failed Transactions

1. Check the error code
2. Fix the underlying issue (if possible)
3. Create a new quote
4. Execute the new quote
5. Original transaction remains FAILED for audit trail

## Idempotency

All Grid API endpoints support idempotency. Include an `Idempotency-Key` header:

```bash
curl -X POST https://api.lightspark.com/grid/2025-10-13/quotes \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-key-123" \
  -d '{...}'
```

This ensures that if you retry a request due to a network error, the same quote/transaction is returned instead of creating a duplicate.
