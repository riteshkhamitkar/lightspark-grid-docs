# Build Your Flow

Design your flow and get the API calls for your exact use case.

## Flow Builder Overview

The Flow Builder (https://docs.lightspark.com/flow-builder) is an interactive tool that lets you:

1. **Select a source** - Where funds are coming from
2. **Select a destination** - Where funds are going
3. **Choose payment rails** - How funds will be routed
4. **Get API calls** - Auto-generated cURL for your flow

## Supported Flows

### US Dollar (USD) to Indian Rupee (INR)

**Source:** United States Dollar (USD)
**Source Rail:** RTP (Real-Time Payments)

**Destination:** Indian Rupee (INR)
**Destination Rail:** UPI (Unified Payments Interface)

**Flow:**
1. Register customer on Grid
2. Complete KYC verification
3. Fund USD internal account
4. Add beneficiary UPI account
5. Create quote (USD -> INR)
6. Execute quote
7. Funds delivered via UPI

### US Dollar (USD) to Mexican Peso (MXN)

**Source:** United States Dollar (USD)
**Destination:** Mexican Peso (MXN)
**Destination Rail:** SPEI/CLABE

### US Dollar (USD) to Brazilian Real (BRL)

**Source:** United States Dollar (USD)
**Destination:** Brazilian Real (BRL)
**Destination Rail:** PIX

### US Dollar (USD) to Philippine Peso (PHP)

**Source:** United States Dollar (USD)
**Destination:** Philippine Peso (PHP)
**Destination Rail:** InstaPay

### USD to Bitcoin (BTC)

**Source:** United States Dollar (USD)
**Destination:** Bitcoin (BTC)
**Destination Rail:** Lightning Network

### USD to UMA Address

**Source:** United States Dollar (USD)
**Destination:** UMA Address
**Destination Rail:** UMA Network

## Example: USD to INR Flow

### Step 1: Register Customer

```bash
curl -X POST https://api.lightspark.com/grid/2025-10-13/customers \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "cust_12345",
    "fullName": "Jane Doe",
    "birthDate": "1990-05-15",
    "nationality": "US",
    "region": "US",
    "email": "jane@example.com",
    "address": {
      "line1": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postalCode": "94105",
      "country": "US"
    }
  }'
```

### Step 2: Generate KYC Link

```bash
curl -X POST https://api.lightspark.com/grid/2025-10-13/customers/{customerId}/kyc-link \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

### Step 3: Fund Account

```bash
# Sandbox
curl -X POST https://api.lightspark.com/grid/2025-10-13/sandbox/internal-accounts/{accountId}/fund \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "1000.00",
    "currency": "USD"
  }'
```

### Step 4: Add UPI External Account

```bash
curl -X POST https://api.lightspark.com/grid/2025-10-13/customers/{customerId}/external-accounts \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "UPI",
    "upiId": "recipient@upi",
    "currency": "INR",
    "accountOwnerName": "Recipient Name"
  }'
```

### Step 5: Create Quote

```bash
curl -X POST https://api.lightspark.com/grid/2025-10-13/quotes \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:usd-123"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "ExternalAccount:upi-456"
    },
    "amount": "100.00",
    "amountCurrency": "USD"
  }'
```

### Step 6: Execute Quote

```bash
curl -X POST https://api.lightspark.com/grid/2025-10-13/quotes/{quoteId}/execute \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json"
```

## Using the Flow Builder

1. Go to https://docs.lightspark.com/flow-builder
2. Select your source currency and rail
3. Select your destination currency and rail
4. Copy the generated cURL commands
5. Adapt them to your use case

## Common Flow Patterns

### Pattern 1: Prefunded Payout
```
[Fund Internal Account] -> [Create Quote] -> [Execute Quote] -> [Delivered]
```

### Pattern 2: JIT Funded Payout
```
[Create Quote (JIT)] -> [Get Payment Instructions] -> [User Sends Funds] -> [Auto Execute]
```

### Pattern 3: Global Account Withdrawal
```
[Create Quote] -> [Get payloadToSign] -> [User Signs] -> [Execute with Signature] -> [Delivered]
```

### Pattern 4: UMA Payment
```
[Create Quote to UMA Address] -> [Execute Quote] -> [Instant Delivery]
```

## Tips

- Use `Copy all` to get all API calls at once
- Switch between cURL and SDK code
- Use the AI agent option for natural language queries
- Hover on the flow diagram for details
