# Ramps Quickstart

Complete guide for converting fiat to crypto (on-ramp) and delivering Bitcoin to a Spark wallet.

## Setup

```bash
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
export GRID_CLIENT_ID="YOUR_SANDBOX_CLIENT_ID"
export GRID_CLIENT_SECRET="YOUR_SANDBOX_CLIENT_SECRET"
```

## On-Ramp: Fiat to Crypto

### Step 1: Create Customer

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "ramp-001",
    "region": "US",
    "fullName": "Crypto Buyer",
    "birthDate": "1990-01-15",
    "nationality": "US",
    "email": "buyer@example.com"
  }'
```

### Step 2: Fund Internal Account

```bash
# Sandbox: Direct fund
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:.../fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"amount": "1000.00", "currency": "USD"}'
```

### Step 3: Add Crypto Wallet (External Account)

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "LIGHTNING_SPARK",
    "address": "$user@wallet.com",
    "currency": "BTC",
    "accountOwnerName": "Crypto Buyer"
  }'
```

### Step 4: Create Quote (USD to BTC)

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
      "accountId": "ExternalAccount:btc-wallet-456"
    },
    "amount": "100.00",
    "amountCurrency": "USD"
  }'
```

### Step 5: Execute Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:.../execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json"
```

## Off-Ramp: Crypto to Fiat

### Step 1: Have BTC Balance

Customer needs BTC in their internal account or wallet.

### Step 2: Add Bank Account

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "ACH",
    "accountNumber": "1234567890",
    "routingNumber": "021000021",
    "currency": "USD",
    "accountOwnerName": "Crypto Seller"
  }'
```

### Step 3: Create Quote (BTC to USD)

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
    "amount": "0.001",
    "amountCurrency": "BTC"
  }'
```

### Step 4: Execute

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:.../execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json"
```

## Complete Flow Summary

```
On-Ramp (Fiat -> Crypto):
1. Create Customer
2. Fund USD account
3. Add BTC wallet
4. Create quote USD->BTC
5. Execute quote
6. BTC arrives in wallet

Off-Ramp (Crypto -> Fiat):
1. Have BTC balance
2. Add bank account
3. Create quote BTC->USD
4. Execute quote
5. USD arrives in bank account
```
