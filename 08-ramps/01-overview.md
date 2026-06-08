# Ramps

Seamlessly convert between fiat currencies and cryptocurrencies through a single, simple API.

Grid automatically handles currency conversion, compliance, and instant settlement via the Lightning Network.

## Features

### Instant Conversion
Convert fiat to crypto (on-ramp) or crypto to fiat (off-ramp) in seconds using real-time exchange rates.

### Simplified Infrastructure
Grid handles the conversion and settlement process, reducing complexity in your crypto integration.

### Global Settlement
Leverages the Lightning Network for instant Bitcoin transfers and local banking rails for fiat settlement worldwide.

## How Ramps Work

1. **Quote Creation** - Get real-time exchange rates and payment instructions for your desired conversion (fiat to crypto or crypto to fiat).
2. **Funding & Conversion** - Fund the quote using the provided payment instructions. Grid executes the conversion at the quoted rate.
3. **Settlement** - Receive your converted funds via Lightning Network (for crypto) or local banking rails (for fiat) within seconds.

## Use Cases

- **Crypto On-Ramp** - Let users buy Bitcoin with their bank account
- **Crypto Off-Ramp** - Let users sell Bitcoin to their bank account
- **Embedded Finance** - Add crypto buying/selling to any app
- **Cross-Border Crypto** - Send crypto internationally

## Supported Conversions

| From | To | Notes |
|------|-----|-------|
| USD | BTC | On-ramp |
| BTC | USD | Off-ramp |
| USD | USDC | Stablecoin on-ramp |
| USDC | USD | Stablecoin off-ramp |
| EUR | BTC | On-ramp |
| BTC | EUR | Off-ramp |

## Prerequisites

- Grid platform with crypto currencies enabled
- API credentials
- Customer KYC completed
- Understanding of quote and execute flow
