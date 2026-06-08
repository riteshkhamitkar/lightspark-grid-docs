# Self-Custody Wallet Integration

Send and receive cryptocurrency to and from user-controlled wallets.

## What is Self-Custody?

Self-custody means users control their own private keys. Grid facilitates the transfer but doesn't hold the user's crypto.

## Supported Wallets

- **Spark Wallet** (spark.money) - Grid's native wallet
- **Any Lightning-compatible wallet**
- **UMA-enabled wallets**

## Sending to External Wallets

### Lightning Address

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
      "accountId": "ExternalAccount:lightning-456"
    },
    "amount": "0.001",
    "amountCurrency": "BTC"
  }'
```

### UMA Address

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
      "umaAddress": "$receiver@wallet.com"
    },
    "amount": "50.00",
    "amountCurrency": "USD"
  }'
```

## Receiving from External Wallets

Users can send crypto to their Grid internal account:

1. Get the account's deposit address:
```bash
curl -X GET "$GRID_BASE_URL/internal-accounts/InternalAccount:btc-123" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

2. User sends crypto to the provided address
3. Grid detects and credits the internal account
4. `incoming_payment.received` webhook sent

## Wallet Security

Grid never stores user's private keys. Users are responsible for:
- Securing their seed phrase
- Backing up their wallet
- Protecting against loss/theft

## Best Practices

1. **Validate addresses** - Check format before sending
2. **Start small** - Test with small amounts
3. **Show network** - Confirm correct network (Bitcoin, Lightning)
4. **Educate users** - Explain self-custody responsibility
