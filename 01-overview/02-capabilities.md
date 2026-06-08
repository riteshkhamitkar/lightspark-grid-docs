# Capabilities

Grid exposes low-level primitives that you combine to build any payment flow. Fiat, crypto, or both.

## Send
Push funds to bank accounts in 65+ countries, crypto wallets, or UMA addresses. You choose the destination, Grid handles delivery.

- **Local Rails:** ACH, SEPA, SEPA Instant, PIX, UPI, CLABE, Bank Transfer, and more
- **Crypto Rails:** Bitcoin Lightning Network, stablecoins (USDC, USDB)
- **UMA Addresses:** Universal Money Addresses for instant global transfers
- **Coverage:** 60+ countries for local rail payouts

## Receive
Accept incoming payments via bank transfer, crypto deposit, or UMA. Webhooks notify you in real-time.

- **Bank Transfers:** ACH push, SEPA, wire transfers
- **Crypto Deposits:** Bitcoin, stablecoins
- **UMA Payments:** Receive via `$user@domain` addresses
- **Real-time Notifications:** Webhook events for all payment status changes

## Convert
Exchange between fiat currencies or convert fiat to crypto. Lock in rates before execution.

- **Fiat-to-Fiat:** USD to EUR, BRL to MXN, and more
- **Fiat-to-Crypto:** USD to BTC, USD to USDC
- **Crypto-to-Fiat:** BTC to USD, USDC to local currency
- **Rate Locking:** Quotes lock rates for 1-15 minutes
- **Transparent Fees:** Full fee breakdown in every quote

## Hold
Maintain balances in multiple currencies. Fund and manage accounts via API.

### Supported Balance Currencies
- **Fiat:** USD, EUR, BRL, MXN, PHP, and more
- **Stablecoins:** USDC, USDB (Brale-issued)
- **Crypto:** BTC

### Account Types
- **Platform Internal Accounts:** Your master accounts for operations
- **Customer Internal Accounts:** Individual customer balances
- **Global Accounts:** Self-custody embedded wallets for customers

## Ramp
On-ramp from bank accounts to BTC or stablecoins. Off-ramp crypto to local bank rails instantly.

- **On-ramp:** Fiat from bank account to crypto wallet
- **Off-ramp:** Crypto to local bank account
- **Instant Conversion:** Real-time rates, atomic execution
- **Self-Custody:** Users control their own keys

## Bill
Generate payment requests and collect funds. Track payment status through completion.

*Note: Coming soon*

## Program
Automate flows with webhooks, approval logic, and idempotent operations. Build any payment experience.

- **Webhooks:** Real-time event notifications
- **Idempotency:** Safe retry logic for all operations
- **Agents:** AI agent integration with policy controls (experimental)
- **Approval Flows:** Multi-step approval for large transactions

## Identify
KYC/KYB verification - use Grid's hosted flows or bring your own compliance.

- **Hosted KYC:** SumSub-powered identity verification
- **KYB Verification:** Business verification with beneficial owners
- **Document Upload:** Support for ID documents, proof of address
- **Status Tracking:** Real-time webhook notifications for status changes

## Capability Matrix by Product

| Capability | Payouts & B2B | Ramps | Rewards | Global P2P | Cards | Global Accounts |
|-----------|--------------|-------|---------|------------|-------|-----------------|
| Send | Yes | Yes | Yes | Yes | - | Yes |
| Receive | Yes | Yes | - | Yes | - | Yes |
| Convert | Yes | Yes | Yes | Yes | - | Yes |
| Hold | Yes | Yes | Yes | Yes | - | Yes |
| Ramp | - | Yes | - | - | - | - |
| Identify | Yes | Yes | Yes | Yes | Yes | Yes |
