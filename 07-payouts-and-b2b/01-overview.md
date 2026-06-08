# Payouts & B2B

Send and receive low-cost real-time payments to bank accounts worldwide through a single, simple API.

Grid automatically routes each payment across its network of Grid switches, handling FX, blockchain settlement, and instant banking off-ramps for you.

## Features

### Instant Conversion
Single API, global reach. Grid interacts with the Money Grid to route your payments globally.

### Simplified Infrastructure
No crypto handling. Grid converts between fiat and crypto instantly to simplify your implementation and minimize FX costs.

### Global Settlement
Real-time settlement. Leverages local instant banking rails and global low latency crypto rails to settle payments in real-time.

## Payment Flow

1. **Fund Internal Account or prepare for Just-in-Time Funding**
   - You can either prefund an internal account with fiat or receive just-in-time payment instructions as part of the quote.

2. **Create Quote**
   - Create a quote to lock exchange rate to the receiving foreign account and parse payment instructions.

3. **Execute Quote**
   - Execute the quote or send funds as per payment instructions to initiate the transfer from the internal account to the external bank account.

## Use Cases

- **Accounts Payable Automation** - Pay international vendors and suppliers
- **Payroll** - Pay remote workers globally
- **Insurance Claims** - Instant beneficiary payouts
- **Freelance Platforms** - Pay contractors in their local currency
- **Marketplace Payouts** - Pay sellers worldwide
- **Cross-Border B2B** - Business-to-business international payments

## Prerequisites

- Grid platform account
- API credentials
- Webhook endpoint configured
- At least one funding currency configured
