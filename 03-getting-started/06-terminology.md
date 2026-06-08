# Core Concepts & Terminology

Core concepts and terminology for the Grid API.

## Key Terms

### Platform
Your business entity registered with Grid. Holds API credentials, configuration, and owns all customers and accounts.

### Customer
A person (individual) or business entity that uses your platform's payment services. Customers must complete KYC (individuals) or KYB (businesses) verification.

### KYC (Know Your Customer)
Identity verification process for individuals. Includes document verification, identity checks, and sanctions screening.

### KYB (Know Your Business)
Verification process for business entities. Includes business registration verification, beneficial owner identification, and compliance checks.

### Internal Account
A Grid-managed account that holds funds within the platform. Used for storing balances and sending payments.

### External Account
A connected bank account or wallet outside of Grid. Used as payment destinations for payouts.

### Global Account
A self-custody embedded wallet account that gives customers branded dollar accounts with customer-controlled transaction signing.

### Quote
A locked-in exchange rate with transparent fee calculations. Quotes expire after a set time (typically 1-5 minutes).

### Transaction
A record of a payment or transfer between accounts.

### UMA (Universal Money Address)
A protocol for instant, low-cost global payments using email-like addresses: `$user@domain.com`.

### Payment Rail
The network or system used to move funds (e.g., ACH, SEPA, PIX, Lightning Network).

### Beneficial Owner
An individual who owns 25% or more of a business entity, directly or indirectly. Must be verified as part of KYB.

### Control Person
An individual with significant responsibility to control, manage, or direct a legal entity (e.g., CEO, CFO).

### SumSub
The third-party identity verification provider used by Grid for KYC/KYB processes.

### Embedded Wallet
A self-custody cryptocurrency wallet provisioned by Grid for Global Accounts. Customers control their own keys.

### Spark
A Lightning-compatible Bitcoin L2 (Layer 2) that supports instant, low-fee Bitcoin and stablecoin transfers.

### USDB
A Brale-issued stablecoin used in Global Accounts. 1 USDB = 1 USD.

### JIT Funding (Just-In-Time)
A funding model where payment instructions are provided in the quote, and the customer sends funds to complete the payment.

### Prefunded
A funding model where funds are already held in an internal account before creating a quote.

### Idempotency
The property of an operation where executing it multiple times has the same effect as executing it once. Grid supports idempotency keys for safe retries.

### Webhook
An HTTP callback sent by Grid to your server to notify you of events.

### Sandbox
A testing environment that simulates all API behavior without moving real money.

### Magic Values
Special test values used in sandbox to trigger specific API responses (e.g., KYC approval/rejection).

## Abbreviations

| Abbreviation | Meaning |
|-------------|---------|
| ACH | Automated Clearing House (US bank transfers) |
| SEPA | Single Euro Payments Area |
| PIX | Brazilian instant payment system |
| UPI | Unified Payments Interface (India) |
| CLABE | Clave Bancaria Estandarizada (Mexico) |
| IBAN | International Bank Account Number |
| FX | Foreign Exchange |
| KYB | Know Your Business |
| KYC | Know Your Customer |
| AML | Anti-Money Laundering |
| PEP | Politically Exposed Person |
| API | Application Programming Interface |
| SDK | Software Development Kit |
| CLI | Command Line Interface |
| JWT | JSON Web Token |
| OIDC | OpenID Connect |
| OTP | One-Time Password |
| PAN | Primary Account Number (card number) |
| CVV | Card Verification Value |
| FPS | Faster Payments System (UK) |
| RTP | Real-Time Payments (US) |
| L2 | Layer 2 (blockchain scaling) |
