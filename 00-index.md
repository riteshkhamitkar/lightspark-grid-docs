# Lightspark Grid API Documentation - Complete Reference

> **Last Updated:** 2026-06-08 | **Source:** https://docs.lightspark.com/

## Table of Contents

This documentation folder contains a complete, structured reference for integrating with **Lightspark Grid** - a low-level payment infrastructure API for global money movement across fiat, stablecoins, and Bitcoin.

### Documentation Sections

| # | Section | Description |
|---|---------|-------------|
| 01 | [Overview](./01-overview/) | What is Grid, capabilities, use cases, FAQ |
| 02 | [Core Concepts](./02-core-concepts/) | Entities, quotes, accounts, transactions, currencies |
| 03 | [Getting Started](./03-getting-started/) | Auth, environments, sandbox, SDKs, webhooks |
| 04 | [KYC/KYB & SumSub](./04-kyc-kyb-sumsub/) | Identity verification, hosted KYC, beneficial owners |
| 05 | [Customers](./05-customers/) | Customer management API reference |
| 06 | [Global Accounts](./06-global-accounts/) | Self-custody dollar accounts, auth, signing |
| 07 | [Payouts & B2B](./07-payouts-and-b2b/) | Cross-border payouts, B2B payments |
| 08 | [Ramps](./08-ramps/) | Fiat-to-crypto and crypto-to-fiat conversion |
| 09 | [Rewards](./09-rewards/) | Bitcoin rewards distribution |
| 10 | [Global P2P](./10-global-p2p/) | Peer-to-peer payments via UMA |
| 11 | [Cards](./11-cards/) | Virtual debit card issuance |
| 12 | [API Reference](./12-api-reference/) | Complete API endpoint documentation |
| 13 | [Webhooks](./13-webhooks/) | All webhook events and handling |
| 14 | [Build Your Flow](./14-build-your-flow/) | Flow builder guide for common patterns |
| 99 | [End-to-End Flow](./99-end-to-end-flow/) | Complete integration guide with real examples |

## Key Capabilities

- **Send** - Push funds to bank accounts in 65+ countries, crypto wallets, or UMA addresses
- **Receive** - Accept incoming payments via bank transfer, crypto deposit, or UMA
- **Convert** - Exchange between fiat currencies or convert fiat to crypto with locked rates
- **Hold** - Maintain balances in USD, EUR, BRL, BTC, USDC, and more
- **Ramp** - On-ramp from bank accounts to BTC or stablecoins; off-ramp crypto to local rails
- **Bill** - Generate payment requests and collect funds
- **Program** - Automate flows with webhooks, approval logic, and idempotent operations
- **Identify** - KYC/KYB verification with hosted flows

## API Base URLs

| Environment | Base URL |
|-------------|----------|
| Sandbox | `https://api.lightspark.com/grid/2025-10-13` |
| Production | `https://api.lightspark.com/grid/2025-10-13` |

## Authentication

All API requests use HTTP Basic Authentication:
- **Username:** Your Client ID
- **Password:** Your Client Secret

```bash
curl -u "CLIENT_ID:CLIENT_SECRET" https://api.lightspark.com/grid/2025-10-13/...
```

## Quick Start

1. Create account at [app.lightspark.com](https://app.lightspark.com)
2. Generate Sandbox API Keys
3. Make your first API call

## Official Resources

- **Docs:** https://docs.lightspark.com/
- **GitHub:** https://github.com/lightsparkdev/grid-api
- **Dashboard:** https://app.lightspark.com
- **OpenAPI Spec:** https://docs.lightspark.com/openapi.yaml
- **LLMs.txt:** https://docs.lightspark.com/llms.txt

---

*This documentation is compiled from the official Lightspark Grid documentation for LLM reference and integration purposes.*
