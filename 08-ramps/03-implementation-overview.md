# Ramps Implementation Overview

High-level guide for implementing fiat-to-crypto and crypto-to-fiat conversion flows.

## Architecture

```
User -> Your App -> Grid API -> Lightning Network / Bank Rails
```

## Implementation Steps

### Phase 1: Setup

1. Configure platform with BTC, USD, USDC currencies
2. Set up webhook endpoint
3. Implement quote display UI

### Phase 2: Customer Onboarding

Same as Payouts:
1. Create customer
2. Complete KYC
3. Set up internal accounts

### Phase 3: Wallet Integration

For on-ramps:
1. Collect user's crypto wallet address
2. Create external account with LIGHTNING_SPARK type
3. Validate address format

For off-ramps:
1. Collect user's bank account details
2. Create external account (ACH, SEPA, etc.)
3. Verify account details

### Phase 4: Conversion Flow

1. Display quote to user (rate, fees, total)
2. Get confirmation
3. Execute quote
4. Show transaction status
5. Notify on completion

## UI Considerations

### Quote Display

Show clearly:
- Amount sending
- Amount receiving
- Exchange rate
- Fees
- Total cost
- Estimated arrival time

### Status Tracking

Show progress:
- Quote created
- Awaiting funds
- Converting
- Sending
- Completed

## Error Handling

| Scenario | Handling |
|----------|----------|
| Rate changed | Create new quote |
| Insufficient funds | Prompt to add funds |
| Invalid address | Validate before quote |
| Network error | Retry with backoff |
| Conversion failed | Refund or retry |
