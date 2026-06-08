# Global Accounts

Give your customers a branded, self-custody dollar account powered by Grid.

## What is a Grid Global Account?

A Grid Global Account is powered by a self-custody embedded Spark wallet that Grid provisions for your customer. It holds a stablecoin or BTC balance and participates in the standard Grid payment flows.

It behaves like any other internal account for incoming funds, but every outbound transfer must be authorized by the customer - a session signing key issued for their device signs each payment.

In the API, a Global Account is an internal account with `type: "EMBEDDED_WALLET"` that participates in the standard Grid customer, quote, transaction, and webhook flows.

## Why a Grid Global Account?

### Self-Custody
Grid never has unilateral access to move user funds, and neither do you. The customer's device is the only party that can authorize a transaction.

### Stablecoin-Denominated
Balances are held as stablecoins like Brale-issued USDB. Use the standard `/quotes` API to convert in from fiat or out to any supported Grid bank-account rail (ACH, PIX, CLABE, UPI, IBAN, UMA, ...).

### Grid-Native
You reuse the customer, internal-account, quote, transaction, and webhook primitives you already integrated for Payouts or P2P. The only thing that's new is an auth + signing layer at the account.

### Built on Bitcoin
Global Accounts run on Spark, a Lightning-compatible Bitcoin L2 that supports instant, low-fee Bitcoin and Stablecoin transfers. You get the benefits of running on Bitcoin, the most neutral, decentralized, and secure network for money.

## Payment Flow

Grid Global Accounts ride on the same `/quotes` + `/quotes/{id}/execute` pattern as every other Grid payment. The only thing that is different is that outbound transfers need a client signature.

### Incoming Funds
Funding an account works like any other internal account. Create a quote with the Global Account as the `destination`, execute it, and Grid converts the source currency into USDB and credits the account. No customer approval needed - incoming value is passive.

### Outgoing Funds
Withdrawals and transfers out require the customer to authorize them on their device. Grid returns a `payloadToSign` in the quote's `paymentInstructions`; the client signs those bytes with its session signing key and passes the base64 signature as the `Grid-Wallet-Signature` header on `/quotes/{id}/execute`. Only then does Grid release the funds.

## Key Components

1. **Authentication** - Passkey, OAuth, or email OTP registration
2. **Client Keys & Signing** - P-256 key pair generation and signing
3. **Sessions** - Authentication session management
4. **Exporting** - Allow customers to export their wallet seed

## Use Cases

- Financial apps and wallets with branded dollar accounts
- Creator platforms for payouts and audience earnings
- Marketplaces for seller earnings
- On-demand platforms for gig worker payments
- Messaging platforms for in-chat transfers
- Social platforms for creator earnings

## Prerequisites

- Platform configured with USDB in supported currencies
- Sandbox or production API credentials
- Understanding of Grid's quote and transaction system
