# Sending Payments

Learn how to send payments from internal accounts to external bank accounts with same-currency and cross-currency transfers.

## Same-Currency Transfer

When source and destination currencies match:

```bash
POST /transfer-outs
```

```bash
curl -X POST "$GRID_BASE_URL/transfer-outs" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "Customer:...",
    "sourceAccountId": "InternalAccount:usd-123",
    "destinationAccountId": "ExternalAccount:ach-456",
    "amount": "1000.00",
    "currency": "USD"
  }'
```

## Cross-Currency Transfer (Quote + Execute)

When converting between currencies:

### Step 1: Create Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:usd-123"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "ExternalAccount:spei-456"
    },
    "amount": "500.00",
    "amountCurrency": "USD"
  }'
```

### Step 2: Execute Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:.../execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json"
```

## UMA Payment

Send to a UMA address:

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:usd-123"
    },
    "destination": {
      "destinationType": "UMA_ADDRESS",
      "umaAddress": "$receiver@domain.com"
    },
    "amount": "100.00",
    "amountCurrency": "USD"
  }'
```

## Transfer-In (Receiving Funds)

Transfer from an external account to an internal account:

```bash
curl -X POST "$GRID_BASE_URL/transfer-ins" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "Customer:...",
    "sourceAccountId": "ExternalAccount:ach-456",
    "destinationAccountId": "InternalAccount:usd-123",
    "amount": "1000.00",
    "currency": "USD"
  }'
```

## Payment Timing by Rail

| Rail | Settlement Time |
|------|----------------|
| ACH | 1-3 business days |
| Wire | Same day |
| SEPA Instant | Seconds |
| SEPA | Same day |
| SPEI/CLABE | Same day |
| PIX | Seconds |
| UPI | Seconds |
| InstaPay | Minutes |
| UMA | Seconds |

## Idempotency

Use idempotency keys to prevent duplicate payments:

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: payout-001-20250115" \
  -d '{...}'
```

## Error Handling

| Error | Cause | Solution |
|-------|-------|----------|
| INSUFFICIENT_FUNDS | Low balance | Fund account |
| INVALID_ACCOUNT | Account closed/inactive | Verify account |
| QUOTE_EXPIRED | Quote timeout | Create new quote |
| BANK_REJECTED | Bank returned | Verify details |
| LIMIT_EXCEEDED | Over limit | Request increase |

## Best Practices

1. **Prefund for speed** - Keep accounts funded for instant payouts
2. **Show quotes** - Let users review before executing
3. **Handle expiration** - Quotes expire in minutes
4. **Monitor status** - Track via webhooks
5. **Use idempotency** - Prevent duplicates
6. **Log everything** - Keep audit trail
