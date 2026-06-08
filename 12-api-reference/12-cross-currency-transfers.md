# Cross-Currency Transfers API

## Create a Transfer Quote

```bash
POST /quotes
```

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
    "amount": "100.00",
    "amountCurrency": "USD"
  }'
```

## Request Parameters

| Field | Type | Required |
|-------|------|----------|
| `source` | object | Yes |
| `source.sourceType` | string | Yes (`ACCOUNT` or `JIT`) |
| `source.accountId` | string | If sourceType=ACCOUNT |
| `destination` | object | Yes |
| `destination.destinationType` | string | Yes (`ACCOUNT` or `UMA_ADDRESS`) |
| `destination.accountId` | string | If destinationType=ACCOUNT |
| `destination.umaAddress` | string | If destinationType=UMA_ADDRESS |
| `amount` | string | Yes |
| `amountCurrency` | string | Yes |

## Execute a Quote

```bash
POST /quotes/{id}/execute
```

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:.../execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -H "Grid-Wallet-Signature: base64_sig"  # For Global Accounts
```

## Get Quote by ID

```bash
GET /quotes/{id}
```

## Estimate Crypto Withdrawal Fee

```bash
GET /quotes/estimate-crypto-fee
```

## Look Up External Account for Payment

```bash
GET /external-accounts/{id}/lookup
```

## Look Up UMA Address for Payment

```bash
GET /uma-addresses/{uma_address}/lookup
```
