# Depositing Funds

How to fund internal accounts for payouts.

## Methods

### 1. Bank Transfer (ACH Push / Wire)

Use the payment instructions from the internal account:

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts/InternalAccount:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Response includes:
```json
{
  "paymentInstructions": {
    "accountNumber": "1234567890",
    "routingNumber": "021000021",
    "bankName": "Lightspark Bank",
    "reference": "PLATFORM_CUST_001"
  }
}
```

Send an ACH push or wire from your bank using these details. Include the reference number.

### 2. Sandbox Funding

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:.../fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "50000.00",
    "currency": "USD"
  }'
```

### 3. Transfer from Another Internal Account

Create a quote between two internal accounts:

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:platform-master"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "InternalAccount:customer-001"
    },
    "amount": "1000.00",
    "amountCurrency": "USD"
  }'
```

## Timing

| Method | Timing |
|--------|--------|
| ACH Push | 1-3 business days |
| Wire | Same day |
| Internal Transfer | Instant |
| Sandbox | Instant |

## Monitoring Deposits

### Webhook

```json
{
  "eventType": "incoming_payment.received",
  "data": {
    "accountId": "InternalAccount:...",
    "amount": "50000.00",
    "currency": "USD"
  }
}
```

### API Polling

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts/InternalAccount:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Best Practices

1. **Reference numbers** - Always include the reference
2. **Confirm receipt** - Wait for webhook or check balance
3. **Reconcile** - Match deposits with bank statements
4. **Alert on delays** - Follow up if deposit not received
5. **Maintain buffer** - Keep sufficient balance
