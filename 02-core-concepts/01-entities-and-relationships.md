# Entities & Relationships

Understanding Grid's core data model is essential for building integrations.

## Core Entities

### Platform
The top-level entity representing your business. Your platform:
- Has API credentials (Client ID and Client Secret)
- Is configured with supported currencies
- Has webhook endpoints for receiving events
- Owns all customers and accounts
- Can hold platform-level internal accounts

### Customer
A person or business that uses your platform's payment services.

**Individual Customer:**
- `customerType`: `INDIVIDUAL`
- Requires KYC verification
- Has internal accounts, external accounts
- Can send and receive payments

**Business Customer:**
- `customerType`: `BUSINESS`
- Requires KYB verification
- Has beneficial owners
- Has internal accounts, external accounts
- Can send and receive payments

**Key Fields:**
- `id`: System-generated unique ID (e.g., `Customer:019542f5-b3e7-1d02-0000-000000000001`)
- `platformCustomerId`: Your platform's identifier for this customer
- `kycStatus`: `PENDING`, `IN_REVIEW`, `APPROVED`, `REJECTED`
- `region`: ISO country code for regulatory jurisdiction
- `currencies`: Array of supported currencies

### Internal Account
Grid-managed accounts that hold funds within the platform.

**Types:**
- `CHECKING`: Standard fiat internal account
- `EMBEDDED_WALLET`: Global Account (self-custody wallet)

**Key Fields:**
- `id`: System-generated unique ID
- `currency`: Account denomination (USD, EUR, BTC, etc.)
- `balance`: Current balance
- `status`: `ACTIVE`, `FROZEN`, `CLOSED`
- `paymentInstructions`: Instructions for funding the account

### External Account
Connected bank accounts and wallets outside of Grid.

**Types by Rail:**
- `ACH`: US bank accounts (account number + routing number)
- `SEPA`: European accounts (IBAN)
- `PIX`: Brazilian accounts (PIX key)
- `UPI`: Indian accounts (UPI ID)
- `CLABE`: Mexican accounts (CLABE number)
- `LIGHTNING_SPARK`: Bitcoin Lightning addresses
- `UMA`: Universal Money Addresses

**Key Fields:**
- `id`: System-generated unique ID
- `accountType`: The payment rail type
- `accountNumber`: Account identifier
- `currency`: Account currency
- `status`: `PENDING`, `ACTIVE`, `VERIFIED`, `FAILED`

### Quote
A locked-in exchange rate with transparent fee calculations.

**Key Fields:**
- `id`: System-generated unique ID
- `source`: Source of funds (internal account or JIT)
- `destination`: Destination account
- `sourceAmount`: Amount to send
- `destinationAmount`: Amount recipient receives
- `exchangeRate`: Locked rate
- `fees`: Total fees
- `expiresAt`: Quote expiration time
- `paymentInstructions`: Instructions for JIT funding

### Transaction
A record of a payment or transfer.

**Key Fields:**
- `id`: System-generated unique ID
- `status`: `PENDING`, `IN_FLIGHT`, `COMPLETED`, `FAILED`, `CANCELLED`
- `type`: `TRANSFER_IN`, `TRANSFER_OUT`, `CONVERSION`
- `amount`: Transaction amount
- `currency`: Transaction currency
- `createdAt`: Creation timestamp
- `completedAt`: Completion timestamp

### Verification
KYC/KYB verification record.

**Key Fields:**
- `id`: System-generated unique ID
- `customerId`: Associated customer
- `verificationStatus`: `RESOLVE_ERRORS`, `PENDING`, `APPROVED`, `REJECTED`
- `errors`: Any missing information
- `kycStatus` / `kybStatus`: Individual/business status

## Entity Relationships

```
Platform
  ├── Customers (INDIVIDUAL / BUSINESS)
  │     ├── Internal Accounts (CHECKING / EMBEDDED_WALLET)
  │     │     ├── Balances (USD, EUR, BTC, USDC, etc.)
  │     │     └── Payment Instructions
  │     ├── External Accounts (ACH, SEPA, PIX, UPI, etc.)
  │     ├── Quotes
  │     ├── Transactions
  │     └── Verifications (KYC/KYB)
  ├── Platform Internal Accounts
  ├── API Tokens
  └── Webhook Configuration
```

## Data Flow

### Typical Payout Flow
```
1. Create Customer
   └── Customer record created with KYC status PENDING

2. Complete KYC
   └── Customer status updated to APPROVED

3. Create Internal Account (auto-created for customer)
   └── Account created with currency based on platform config

4. Fund Internal Account
   └── Balance updated

5. Add External Account (beneficiary)
   └── External account created with status PENDING/ACTIVE

6. Create Quote
   └── Quote generated with locked rate and fees

7. Execute Quote
   └── Transaction created, funds transferred

8. Webhook Notification
   └── Transaction status sent to your webhook endpoint
```

### Global Account Flow
```
1. Create Customer (with USDB in currencies)
   └── Global Account (EMBEDDED_WALLET) auto-provisioned

2. Complete KYC
   └── Customer verified

3. Register Authentication (passkey/OAuth/OTP)
   └── Customer can now sign transactions

4. Fund Global Account
   └── Quote + Execute with internal account as source

5. Create Withdrawal Quote
   └── Quote with payloadToSign for customer signature

6. Customer Signs Transaction
   └── Signature added to execute request

7. Execute Quote with Signature
   └── Funds transferred to external account
```

## API Resource Naming

Grid uses typed IDs for all resources:
- `Customer:...` - Customer entities
- `InternalAccount:...` - Internal accounts
- `ExternalAccount:...` - External accounts
- `Quote:...` - Quotes
- `Transaction:...` - Transactions
- `Verification:...` - Verifications
- `Card:...` - Cards
- `BeneficialOwner:...` - Beneficial owners
- `Document:...` - Documents
