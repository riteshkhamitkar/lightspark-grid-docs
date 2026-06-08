# Quote System

How exchange rates, pricing, and payment execution work.

## What is a Quote?

A quote locks in:
- An exchange rate between two currencies
- Total fees for the transaction
- Exact amounts to be sent and received
- Payment instructions (if JIT funding is needed)
- An expiration time (typically 1-5 minutes, up to 15 minutes depending on the payment type)

Quotes ensure that your customers know exactly what they'll pay and what the recipient will receive before committing to a transaction.

## When Do You Need a Quote?

### Quotes Required
- Cross-currency transfers (USD to EUR, BRL to MXN)
- Fiat-to-crypto conversion (USD to BTC)
- Crypto-to-fiat conversion (BTC to USD)
- UMA payments (always require quotes)
- JIT funded payments (need payment instructions)

### Quotes Optional
- Same-currency transfers where you already know the fee structure

## Creating a Quote

### Basic Cross-Currency Quote

```bash
curl -X POST https://api.lightspark.com/grid/2025-10-13/quotes \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:e85dcbd6-dced-4ec4-b756-3c3a9ea3d965"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "ExternalAccount:8f4e2b1a-9c3d-4e5f-6a7b-8c9d0e1f2a3b"
    },
    "amount": "100.00",
    "amountCurrency": "USD"
  }'
```

### Quote Request Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `source` | object | Yes | Source of funds |
| `source.sourceType` | string | Yes | `ACCOUNT` or `JIT` |
| `source.accountId` | string | Yes* | Internal account ID (if sourceType=ACCOUNT) |
| `destination` | object | Yes | Payment destination |
| `destination.destinationType` | string | Yes | `ACCOUNT` or `UMA_ADDRESS` |
| `destination.accountId` | string | Yes* | External account ID (if destinationType=ACCOUNT) |
| `destination.umaAddress` | string | Yes* | UMA address (if destinationType=UMA_ADDRESS) |
| `amount` | string | Yes | Amount to send |
| `amountCurrency` | string | Yes | Currency of the amount |

### Quote Response

```json
{
  "id": "Quote:a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "sourceAmount": "100.00",
  "sourceAmountCurrency": "USD",
  "destinationAmount": "1675.00",
  "destinationAmountCurrency": "MXN",
  "exchangeRate": "16.75",
  "fees": "2.50",
  "feesCurrency": "USD",
  "totalCost": "102.50",
  "totalCostCurrency": "USD",
  "expiresAt": "2025-01-15T10:05:00Z",
  "paymentInstructions": null,
  "status": "ACTIVE"
}
```

## Locked Currency Side

When creating a quote, you can specify which side to lock:

- **Source amount locked:** You specify exactly how much to send
- **Destination amount locked:** You specify exactly how much the recipient receives

Grid calculates the other side based on the current exchange rate.

## Funding Models

### Prefunded (From Internal Account)

Funds are already in the customer's internal account. The quote debits this account.

**Use case:** Customer has deposited funds, now wants to send a payout.

```json
{
  "source": {
    "sourceType": "ACCOUNT",
    "accountId": "InternalAccount:..."
  }
}
```

### Just-In-Time (JIT) Funding

Customer needs to send funds to complete the payment. The quote includes payment instructions.

**Use case:** Customer wants to send a payout but hasn't prefunded their account.

```json
{
  "source": {
    "sourceType": "JIT"
  }
}
```

JIT quote response includes:
```json
{
  "paymentInstructions": {
    "accountNumber": "...",
    "routingNumber": "...",
    "reference": "...",
    "bankName": "..."
  }
}
```

## Executing a Quote

```bash
curl -X POST https://api.lightspark.com/grid/2025-10-13/quotes/{quote_id}/execute \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json"
```

### For Global Account withdrawals, add signature header:
```bash
curl -X POST https://api.lightspark.com/grid/2025-10-13/quotes/{quote_id}/execute \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -H "Grid-Wallet-Signature: base64_signature_here"
```

## Execution Timing

### Immediate Execution
Most quotes execute immediately when you call the execute endpoint. The transaction is created and funds are transferred right away.

## Fees

Fees are transparent and included in every quote:
- **FX Spread:** The difference between the market rate and the quoted rate
- **Transaction Fee:** A flat fee per transaction
- **Network Fee:** For crypto transactions, the on-chain fee

Total fees are shown in both the source and destination currencies.

## Best Practices

1. **Always show quotes to users before executing** - Let them review rates and fees
2. **Execute quotes before they expire** - Quotes typically expire in 1-5 minutes
3. **Handle quote expiration gracefully** - If a quote expires, create a new one
4. **Store quote IDs** - Quote IDs link to transactions for reconciliation
5. **Monitor quote status** - Check if quotes are ACTIVE, EXPIRED, or EXECUTED

## Quote Lifecycle

```
ACTIVE -> EXECUTED -> COMPLETED
  |
  -> EXPIRED (if not executed within time limit)
```

## Error Handling

Common quote errors:
- `INSUFFICIENT_FUNDS` - Source account doesn't have enough balance
- `QUOTE_EXPIRED` - Quote has expired, create a new one
- `INVALID_ACCOUNT` - Account doesn't exist or is not active
- `UNSUPPORTED_CORRIDOR` - Currency pair not supported
- `LIMIT_EXCEEDED` - Amount exceeds platform or customer limits
