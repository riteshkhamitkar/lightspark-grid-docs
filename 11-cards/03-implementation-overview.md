# Cards Implementation Overview

End-to-end architecture for issuing and operating cards.

## Architecture

```
Cardholder -> Your App -> Grid Cards API -> Card Issuer
                |                              |
                -> Funding Source (Internal Account)
```

## Implementation Steps

### Phase 1: Setup

1. Configure platform for cards
2. Set up webhook endpoint
3. Understand card lifecycle

### Phase 2: Cardholder Onboarding

1. Create customer
2. Complete KYC (status must be APPROVED)
3. Set up internal account as funding source

### Phase 3: Card Issuance

1. Issue card for cardholder
2. Bind funding source
3. Render card credentials via iframe

### Phase 4: Transaction Management

1. Handle authorization webhooks
2. Monitor clearings
3. Handle refunds
4. Reconcile transactions

### Phase 5: Card Operations

1. Freeze/unfreeze cards
2. Update funding sources
3. Close cards

## Security

- PAN/CVV rendered via secure iframe
- Real-time authorization against balance
- Webhook notifications for all activity
