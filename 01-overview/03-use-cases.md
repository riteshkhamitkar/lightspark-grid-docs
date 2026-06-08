# Use Cases

Discover what you can build with Grid across different industries and payment scenarios.

## Global Accounts

### Financial Apps and Wallets
Create dollar accounts users can hold, send from, and withdraw locally.

**Example:** A neobank app where users get a USD account, receive salary deposits, and send money to family abroad via local rails.

### Creator Platforms
Give creators a global dollar account for payouts and audience earnings.

**Example:** A streaming platform where creators earn USD from viewers worldwide and withdraw to their local bank accounts.

### Marketplaces
Let sellers collect marketplace earnings and withdraw through local rails.

**Example:** An e-commerce platform where international sellers receive USD from sales and withdraw via PIX in Brazil or UPI in India.

### On-Demand Platforms
Pay gig workers into dollar accounts they can withdraw from locally.

**Example:** A ride-sharing app paying drivers in USD to their Global Account, which they can withdraw to local bank accounts instantly.

### Messaging Platforms
Add dollar accounts for transfers inside chat and community experiences.

### Social Platforms
Help creators and communities receive, hold, and move dollar balances globally.

## Payouts & B2B

### Accounts Payable Automation
Businesses paying international vendors, freelancers, and suppliers.

**Flow:**
1. Business customer (your client) funds their internal account
2. They initiate payouts to vendors through your platform
3. Grid converts USD to local currency and delivers via local rails

### Payroll
Pay remote workers and contractors globally.

### Supplier Payments
Pay international suppliers in their local currency.

### Insurance Claims
Instant payout of claims to beneficiaries worldwide.

## Ramps

### Crypto On-Ramp
Let users buy crypto with their bank account.

**Flow:**
1. User connects bank account
2. Creates quote for USD to BTC
3. Funds transfer via ACH
4. Receives BTC in self-custody wallet

### Crypto Off-Ramp
Let users sell crypto to their bank account.

**Flow:**
1. User sends crypto from wallet
2. Grid converts to local fiat
3. Deposits to user's bank account via local rails

### Embedded Finance
Add crypto buying/selling to any app.

## Rewards

### Bitcoin Cashback
Give users Bitcoin rewards for purchases.

### Referral Programs
Pay referral bonuses in Bitcoin instantly.

### Loyalty Points
Convert loyalty points to Bitcoin.

### Gaming Rewards
Pay out gaming winnings in Bitcoin.

**Flow:**
1. Platform holds USD in internal account
2. Creates quote for USD to BTC conversion
3. Executes quote delivering BTC to user's Spark wallet

## Global P2P

### Cross-Border Remittance
Send money person-to-person across borders.

**Example:** A user in the US sends $500 to family in the Philippines:
1. Sender funds their internal account
2. Creates quote for USD to PHP
3. Executes quote - recipient receives PHP via InstaPay

### UMA Payments
Send money using Universal Money Addresses.

**Example:** `$sender@wallet.com` sends to `$receiver@bank.com` - instant, low-fee global transfer.

### Wallet-to-Wallet
Transfer between crypto wallets globally.

## Cards

### Virtual Debit Cards
Issue virtual cards backed by internal accounts.

**Use Cases:**
- Corporate expense cards
- Creator spending accounts
- Marketplace seller cards
- Gig worker payment cards

**Features:**
- Real-time authorization against account balance
- No PAN in your stack (secure iframe rendering)
- Freeze/unfreeze controls
- Automatic reconciliation

## Combining Use Cases

Grid's power comes from combining primitives:

**Example: Creator Platform with Global Accounts + Payouts + Cards**
1. Creator gets Global Account (self-custody USD)
2. Fans tip via UMA (Global P2P)
3. Creator withdraws to local bank (Payouts)
4. Creator gets virtual card for spending (Cards)
5. Platform takes fees via automated splits

**Example: Remittance App with Ramps + Payouts**
1. User on-ramps USD via bank transfer (Ramps)
2. Creates quote for USD to recipient currency (Payouts)
3. Recipient receives in local bank account (Payouts)
4. Both sender and recipient can hold crypto balances
