# Rewards Implementation Overview

High-level implementation guide for Bitcoin rewards.

## Architecture

```
Your Platform
  -> Grid API (Quote + Execute)
    -> FX Conversion (USD -> BTC)
      -> Lightning Network
        -> User's Spark Wallet
```

## Key Entities

| Entity | Role |
|--------|------|
| Platform | Holds USD, initiates rewards |
| Customer | Receives rewards (may be KYC'd or not) |
| External Account | User's BTC wallet address |
| Quote | Locks USD -> BTC rate |
| Transaction | Reward delivery record |

## Implementation Steps

### Phase 1: Setup

1. Fund platform USD internal account
2. Configure webhook endpoint
3. Set up transaction monitoring

### Phase 2: Recipient Management

1. Collect recipient wallet addresses
2. Create external accounts for wallets
3. Validate address formats

### Phase 3: Reward Distribution

1. Create quote (USD -> BTC wallet)
2. Show confirmation (optional)
3. Execute quote
4. Track via webhooks
5. Confirm delivery

### Phase 4: Reporting

1. List transactions
2. Generate reward reports
3. Reconcile balances

## Compliance

For rewards, only the payer (platform or business customer) needs KYB. Individual recipients may not need KYC depending on amounts and jurisdiction. Contact Lightspark for guidance.

## Best Practices

1. **Atomic operation** - Quote + execute handles everything
2. **Idempotency** - Prevent duplicate rewards
3. **Batch processing** - Send multiple rewards efficiently
4. **Monitor delivery** - Track all reward transactions
5. **Handle failures** - Retry failed rewards
