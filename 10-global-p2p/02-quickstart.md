# Global P2P Quickstart

Send your first cross-border payment.

## Setup

```bash
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
export GRID_CLIENT_ID="YOUR_SANDBOX_CLIENT_ID"
export GRID_CLIENT_SECRET="YOUR_SANDBOX_CLIENT_SECRET"
```

## Step 1: Create Sender

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "sender-001",
    "region": "US",
    "fullName": "Sender Name",
    "birthDate": "1990-01-15",
    "nationality": "US",
    "email": "sender@example.com"
  }'
```

## Step 2: Fund Sender Account

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:.../fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"amount": "5000.00", "currency": "USD"}'
```

## Step 3: Send to UMA Address

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

## Step 4: Execute

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:.../execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json"
```

## Send to Bank Account

```bash
# Add receiver's bank account
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "UPI",
    "upiId": "receiver@upi",
    "currency": "INR",
    "accountOwnerName": "Receiver Name"
  }'

# Create quote
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": { "sourceType": "ACCOUNT", "accountId": "InternalAccount:usd-123" },
    "destination": { "destinationType": "ACCOUNT", "accountId": "ExternalAccount:upi-456" },
    "amount": "100.00",
    "amountCurrency": "USD"
  }'
```

## Receive Payments

Set up UMA address for receiving:

```bash
# Customer gets UMA address after creation
# Share this address to receive payments
# e.g., $sender@your-domain.com
```

## Complete Flow Summary

```
UMA Payment:
1. Create sender customer
2. Fund account
3. Create quote (USD -> UMA address)
4. Execute quote
5. Instant delivery to UMA address

Bank Payment:
1. Create sender customer
2. Fund account
3. Add receiver bank account
4. Create quote (USD -> bank account)
5. Execute quote
6. Delivery via local rails
```
