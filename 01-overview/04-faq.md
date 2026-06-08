# FAQ

## What is Grid and who is it for?

Grid is a low-level payment infrastructure API that enables businesses, financial institutions, and developers to send and receive payments globally across fiat currencies, stablecoins, and Bitcoin.

Grid is ideal for:
- Fintech companies building payment products
- Businesses sending international payouts
- Platforms enabling P2P payments
- Companies offering crypto on/off-ramps
- Businesses distributing rewards or incentives

## How do I get started with Grid?

1. **Create an Account** - Go to [app.lightspark.com](https://app.lightspark.com) to create your account and access the Grid dashboard.
2. **Generate Sandbox API Keys** - Generate your sandbox API keys from the dashboard and start building and testing your integration.
3. **Go to Production** - Once testing is complete, reach out to your Lightspark contact to receive production credentials and launch with real transactions.

## Do I need to know about cryptocurrency to use Grid?

No. Grid handles all on-chain operations, wallet management, and crypto conversions behind the scenes. You interact with a simple REST API using familiar concepts like customers, accounts, and transactions. Bitcoin and stablecoins are just another payment rail that Grid manages for you automatically.

## What countries are supported?

Grid supports:
- **60+ countries** for local payment rail payouts
- **Global coverage** for Bitcoin and stablecoin transactions
- **Full platform access** in United States and Europe

See the [Currencies & Payment Rails](../02-core-concepts/05-currencies-and-payment-rails.md) guide for the complete list.

## What currencies are supported?

**Fiat:** USD, EUR, BRL, MXN, PHP, and 50+ more
**Stablecoins:** USDC, USDB
**Crypto:** BTC

## What are the fees?

Fees are transparent and included in every quote. The quote system shows you exactly what you'll pay and what the recipient will receive before you commit to a transaction.

## How does KYC/KYB work?

Grid provides hosted KYC/KYB flows powered by SumSub. You can:
- Generate a hosted KYC link for customers
- Customers complete verification through the hosted flow
- Receive webhook notifications for status changes
- Track verification status via API

See [KYC/KYB documentation](../04-kyc-kyb-sumsub/) for details.

## What is UMA?

UMA (Universal Money Address) is a protocol for instant, low-cost global payments. It works like an email address for money - `$user@domain.com`. Grid supports UMA for both sending and receiving payments.

## What is a Global Account?

A Grid Global Account is a self-custody dollar account powered by Grid. It gives your customers:
- A branded, self-custody account
- Stablecoin-denominated balances (USDB)
- The ability to hold, send, and receive globally
- Customer-controlled transaction signing
- Built on Bitcoin/Spark L2

## How does the quote system work?

Quotes lock in:
- Exchange rate between two currencies
- Total fees for the transaction
- Exact amounts to be sent and received
- Payment instructions (if JIT funding is needed)
- An expiration time (typically 1-5 minutes)

See [Quote System](../02-core-concepts/02-quote-system.md) for full details.

## What are internal and external accounts?

- **Internal Accounts:** Grid-managed accounts that hold funds within the platform. Used for holding balances and sending payments.
- **External Accounts:** Connected bank accounts and wallets outside of Grid. Used as payment destinations.

See [Account Model](../02-core-concepts/03-account-model.md) for full details.

## What is sandbox testing?

The sandbox environment lets you test all API flows without moving real money. Grid provides "magic values" to trigger specific responses like KYC approval/rejection, transfer success/failure, etc.

See [Sandbox Testing](../03-getting-started/03-sandbox-testing.md) for magic values.

## What SDKs are available?

Grid offers official SDKs for multiple languages. See [SDKs & CLI](../03-getting-started/04-sdks-and-cli.md) for details.

## How do webhooks work?

Grid sends webhook notifications for events like:
- Payment status changes
- KYC/KYB status changes
- Customer status changes
- Account balance updates

You configure a webhook URL in your platform settings and Grid sends signed webhook events to that endpoint.

See [Webhooks](../03-getting-started/05-webhooks.md) for details.

## Is Grid regulated?

Grid operates through regulated partners for money transmission and compliance. You should consult with your legal team about regulatory requirements for your specific use case and jurisdiction.

## How do I get support?

- **Documentation:** https://docs.lightspark.com/
- **GitHub Issues:** https://github.com/lightsparkdev/grid-api/issues
- **Contact:** https://www.lightspark.com/contact (Book a live demo)
