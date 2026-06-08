# Cards Quickstart

Issue your first card and simulate a transaction end to end.

## Setup

```bash
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
export GRID_CLIENT_ID="YOUR_SANDBOX_CLIENT_ID"
export GRID_CLIENT_SECRET="YOUR_SANDBOX_CLIENT_SECRET"
```

## Step 1: Create Cardholder

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "cardholder-001",
    "region": "US",
    "fullName": "Card Holder",
    "birthDate": "1990-01-15",
    "nationality": "US",
    "email": "cardholder@example.com"
  }'
```

KYC must be APPROVED before issuing card.

## Step 2: Create Internal Account (Funding Source)

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/.../fund" \
  -d '{"amount": "1000", "currency": "USD"}'
```

## Step 3: Issue Card

```bash
curl -X POST "$GRID_BASE_URL/cards" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "cardholderId": "Customer:cardholder-001",
    "fundingSources": ["InternalAccount:usd-123"],
    "cardName": "My Virtual Card"
  }'
```

## Step 4: Get Card Details

```bash
curl -X GET "$GRID_BASE_URL/cards/Card:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Step 5: Render Card (iframe)

```html
<iframe
  src="https://cards.lightspark.com/render?cardId=Card:...&token=..."
  sandbox="allow-same-origin"
  style="width: 320px; height: 200px; border: none;"
></iframe>
```

## Step 6: Simulate Transaction

```bash
curl -X POST "$GRID_BASE_URL/sandbox/cards/Card:.../authorize" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "50.00",
    "currency": "USD",
    "merchantName": "Test Store"
  }'
```

## Complete Flow

```
1. Create cardholder -> KYC APPROVED
2. Fund internal account
3. Issue card -> Card:... (PENDING_ISSUE -> ACTIVE)
4. Render card via iframe
5. Simulate authorization -> balance checked
6. Simulate clearing -> funds deducted
7. Monitor via webhooks
```
