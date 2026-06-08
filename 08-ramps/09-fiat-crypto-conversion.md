# Fiat-to-Crypto and Crypto-to-Fiat

Build on-ramp and off-ramp flows to convert between fiat currencies and cryptocurrencies.

## On-Ramp (Fiat -> Crypto)

### Flow

1. User wants to buy $100 of BTC
2. Create quote: USD internal account -> BTC external wallet
3. Show quote (rate, fees, BTC amount)
4. User confirms
5. Execute quote
6. BTC sent via Lightning Network

### Example

```bash
# Create quote
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
      "accountId": "ExternalAccount:btc-wallet-456"
    },
    "amount": "100.00",
    "amountCurrency": "USD"
  }'
```

Response:
```json
{
  "id": "Quote:quote-789",
  "sourceAmount": "100.00",
  "sourceAmountCurrency": "USD",
  "destinationAmount": "0.0015",
  "destinationAmountCurrency": "BTC",
  "exchangeRate": "0.000015",
  "fees": "2.50",
  "feesCurrency": "USD",
  "totalCost": "102.50",
  "totalCostCurrency": "USD",
  "expiresAt": "2025-01-15T10:05:00Z"
}
```

## Off-Ramp (Crypto -> Fiat)

### Flow

1. User wants to sell 0.01 BTC for USD
2. Create quote: BTC internal account -> USD bank account
3. Show quote (rate, fees, USD amount)
4. User confirms
5. Execute quote
6. USD sent via ACH/Wire

### Example

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:btc-123"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "ExternalAccount:ach-456"
    },
    "amount": "0.01",
    "amountCurrency": "BTC"
  }'
```

## Slippage Protection

Grid quotes lock in rates for a short period (1-5 minutes). This protects against slippage during execution.

## Price Display

Show users:
- Current rate
- Fee breakdown
- Total amount they'll receive
- Comparison to market rate (if applicable)

## Best Practices

1. **Show all fees** - No hidden costs
2. **Rate comparison** - Show if your rate is competitive
3. **Time limit** - Show quote expiration
4. **Confirm details** - Have user confirm wallet address
5. **Track status** - Show progress after execution
