# Global P2P Implementation Overview

High-level implementation plan for configuring, onboarding, funding, and moving money.

## Architecture

```
Sender -> Your App -> Grid API -> {UMA Network | Local Bank Rails}
```

## Implementation Steps

### Phase 1: Setup

1. Configure platform with supported currencies
2. Set up UMA address domain
3. Configure webhook endpoint

### Phase 2: Onboarding

1. Create sender customer
2. Complete KYC
3. Set up internal account

### Phase 3: Funding

1. Fund internal account
2. Monitor balance

### Phase 4: Sending

1. Collect recipient info (UMA or bank)
2. Create external account (if bank)
3. Create quote
4. Execute quote
5. Track via webhooks

### Phase 5: Receiving

1. Provide UMA address to sender
2. Handle incoming payment webhooks
3. Credit recipient balance

## UMA Address Format

```
$user@your-domain.com
```

Configure your domain in Grid dashboard to enable UMA addresses for your customers.

## Error Handling

| Scenario | Action |
|----------|--------|
| Invalid UMA address | Validate format, check domain |
| UMA not found | Check if receiver is registered |
| Quote expired | Create new quote |
| Bank rejection | Verify account details |

## Best Practices

1. **Validate UMA addresses** - Check format before quote
2. **Show fees upfront** - Transparent cost display
3. **Track in real-time** - Webhook-driven status updates
4. **Handle failures** - Retry with different rails
