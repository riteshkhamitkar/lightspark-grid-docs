# Rewards Quickstart

Complete walkthrough for buying Bitcoin and sending it as a reward to an external Spark wallet for self-custody.

## Setup

```bash
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
export GRID_CLIENT_ID="YOUR_SANDBOX_CLIENT_ID"
export GRID_CLIENT_SECRET="YOUR_SANDBOX_CLIENT_SECRET"
```

## Step 1: Fund Platform Account

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:.../fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "10000.00",
    "currency": "USD"
  }'
```

## Step 2: Create Recipient Customer

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "reward-001",
    "region": "US",
    "fullName": "Reward Recipient",
    "birthDate": "1990-01-15",
    "nationality": "US",
    "email": "recipient@example.com"
  }'
```

In sandbox, KYC is auto-approved.

## Step 3: Add Recipient's Wallet

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "LIGHTNING_SPARK",
    "address": "$recipient@wallet.com",
    "currency": "BTC",
    "accountOwnerName": "Reward Recipient"
  }'
```

## Step 4: Create Reward Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:platform-usd"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "ExternalAccount:recipient-btc"
    },
    "amount": "50.00",
    "amountCurrency": "USD"
  }'
```

Response:
```json
{
  "id": "Quote:reward-789",
  "sourceAmount": "50.00",
  "sourceAmountCurrency": "USD",
  "destinationAmount": "0.00075",
  "destinationAmountCurrency": "BTC",
  "exchangeRate": "0.000015",
  "fees": "1.50",
  "feesCurrency": "USD",
  "totalCost": "51.50",
  "totalCostCurrency": "USD",
  "expiresAt": "2025-01-15T10:05:00Z"
}
```

## Step 5: Execute Reward

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:reward-789/execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json"
```

## Step 6: Monitor

Listen for `transaction.status_change` webhook:

```json
{
  "eventType": "transaction.status_change",
  "data": {
    "transactionId": "Transaction:...",
    "status": "COMPLETED",
    "amount": "0.00075",
    "currency": "BTC",
    "destinationAmount": "0.00075",
    "destinationCurrency": "BTC"
  }
}
```

## Complete Reward Flow

```
1. Fund platform USD account
2. Create recipient customer
3. Add recipient BTC wallet
4. Create quote (USD -> BTC wallet)
5. Execute quote
6. BTC delivered to wallet instantly
7. Webhook notification on completion
```

## Batch Rewards

For sending multiple rewards, loop through recipients:

```javascript
for (const recipient of recipients) {
  const quote = await grid.quotes.create({
    source: { accountId: platformAccountId },
    destination: { accountId: recipient.walletId },
    amount: recipient.amount,
    amountCurrency: 'USD'
  });

  await grid.quotes.execute(quote.id);
}
```

Use idempotency keys to prevent duplicate rewards:
```bash
Idempotency-Key: reward-user123-20250115
```
