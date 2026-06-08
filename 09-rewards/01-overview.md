# Rewards

Send instant Bitcoin rewards to your users' self-custody wallets worldwide through a single, simple API.

Grid automatically handles the entire process, managing the fiat-to-crypto conversion, compliance checks, and instant delivery for you.

## Features

### Buy & Send in One Call
The Grid API combines USD-to-BTC conversion and payout into a single, atomic operation, simplifying the process of distributing rewards.

### No Crypto Handling
Grid converts your platform's fiat balance into Bitcoin on demand, allowing you to offer crypto rewards without managing digital asset custody or complex exchange integrations.

### Instant Delivery
Delivers Bitcoin to your users' wallets in seconds (like a Spark wallet) giving them immediate ownership and control.

## Rewards Payout Flow

1. **Funding** - Your platform's internal account is pre-funded with fiat currency (e.g., USD) via standard payment rails like ACH push.
2. **Customer Onboarding** - For rewards, the only entity who needs to be KYB'd is the entity paying for the reward. This can be you, the platform, or your business customers that want to pay out rewards to their end users.
3. **Quote & Execution** - You execute a single API call to create a quote, instantly convert a specific USD amount to BTC at the current market rate, and transfer the Bitcoin to the user's wallet address.

## Use Cases

- **Bitcoin Cashback** - Give users Bitcoin rewards for purchases
- **Referral Programs** - Pay referral bonuses in Bitcoin instantly
- **Loyalty Points** - Convert loyalty points to Bitcoin
- **Gaming Rewards** - Pay out gaming winnings in Bitcoin
- **Survey Rewards** - Compensate participants with Bitcoin
- **Affiliate Commissions** - Pay affiliates in Bitcoin
- **Contest Prizes** - Distribute prizes in Bitcoin

## Prerequisites

- Grid platform account
- API credentials
- USD internal account funded
- Understanding of quote and execute flow
