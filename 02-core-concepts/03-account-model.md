# Account Model

Internal accounts, external accounts, and how they work together.

Grid uses two types of accounts:
- **Internal accounts** (Grid-managed balances)
- **External accounts** (connected bank accounts and wallets)

## Internal Accounts

Internal accounts are Lightspark managed accounts that hold funds within the Grid platform. They allow you to receive deposits and send payments to external bank accounts or other payment destinations.

They are useful for holding funds on behalf of the platform or customers which will be used for instant, 24/7 quotes and transfers out of the system.

### Types of Internal Accounts

1. **Platform Internal Accounts** - Hold pooled funds for your platform operations (rewards distribution, reconciliation, etc.)
2. **Customer Internal Accounts** - Hold individual customer funds for their transactions
3. **Global Accounts (EMBEDDED_WALLET)** - Self-custody wallets for customers

### How Internal Accounts Work

Internal accounts act as an intermediary holding account in the payment flow:

1. **Deposit funds** - You or your customers deposit money into internal accounts using bank transfers (ACH, wire, PIX, etc.) or crypto transfers
2. **Hold balance** - Funds are held securely in the internal account until needed
3. **Send payments** - You initiate transfers from internal accounts to external destinations

### Internal Account Characteristics

- Each internal account is denominated in a single currency (USD, EUR, BTC, etc.)
- Has a unique balance that you can query at any time
- Includes unique payment instructions for depositing funds
- Supports multiple funding methods depending on the currency

### Creating Internal Accounts

Internal accounts are **automatically created** when you onboard a customer, based on your platform's currency configuration. Platform-level internal accounts are created when you configure your platform with supported currencies.

### Retrieving Internal Accounts

**List customer internal accounts:**
```bash
curl -X GET "https://api.lightspark.com/grid/2025-10-13/internal-accounts?customerId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

**Filter by currency:**
```bash
curl -X GET "https://api.lightspark.com/grid/2025-10-13/internal-accounts?customerId=Customer:...&currency=USD" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

**List platform internal accounts:**
```bash
curl -X GET "https://api.lightspark.com/grid/2025-10-13/internal-accounts?accountOwnerType=PLATFORM" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

### Understanding Funding Payment Instructions

Each internal account includes `paymentInstructions` that tell you how to fund it:

```json
{
  "paymentInstructions": {
    "accountNumber": "1234567890",
    "routingNumber": "021000021",
    "bankName": "Lightspark Bank",
    "reference": "ACCOUNT_ID_REFERENCE",
    "wireInstructions": { ... }
  }
}
```

### Checking Account Balances

```bash
curl -X GET "https://api.lightspark.com/grid/2025-10-13/internal-accounts/InternalAccount:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Response includes:
```json
{
  "id": "InternalAccount:...",
  "currency": "USD",
  "balance": "5000.00",
  "status": "ACTIVE"
}
```

### Displaying Funding Instructions to Customers

To show customers how to fund their account:
1. Get the internal account details
2. Extract `paymentInstructions`
3. Display the relevant details (account number, routing number, reference)
4. Instruct them to include the reference number so funds are credited correctly

### Best Practices for Internal Accounts

1. **Monitor balances** - Keep sufficient funds for expected payouts
2. **Use references** - Always include the reference number in transfers
3. **Reconcile regularly** - Match internal balances with your ledger
4. **Handle multiple currencies** - Create separate accounts per currency

## External Accounts

External accounts are bank accounts, crypto wallets, and payment destinations outside of Grid that you connect for sending payments.

### Characteristics

- Belong to a customer or the platform
- Connected via various payment rails (ACH, SEPA, PIX, UPI, etc.)
- Must be verified before use (in production)
- Can be used as quote destinations

### Supported Account Types

| Rail | Identifier | Countries |
|------|-----------|-----------|
| ACH | Account Number + Routing Number | US |
| WIRE | Account Number + Wire Routing | US |
| SEPA | IBAN | EU |
| SEPA_INSTANT | IBAN | EU |
| PIX | PIX Key | BR |
| UPI | UPI ID | IN |
| CLABE | CLABE Number | MX |
| BANK_TRANSFER | Account Details | Various |
| LIGHTNING_SPARK | Lightning Address | Global |
| UMA | UMA Address | Global |

### Creating External Accounts

```bash
curl -X POST "https://api.lightspark.com/grid/2025-10-13/customers/{customer_id}/external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "ACH",
    "accountNumber": "1234567890",
    "routingNumber": "021000021",
    "currency": "USD",
    "accountOwnerName": "Jane Doe"
  }'
```

### External Account Status

- `PENDING` - Account created, awaiting verification
- `ACTIVE` - Account verified and ready for use
- `VERIFIED` - Additional verification completed
- `FAILED` - Verification failed
- `CLOSED` - Account permanently closed

## Account Combinations

### Pattern 1: Prefunded Payouts

```
[Platform Bank] -> [Customer Internal Account] -> [Beneficiary External Account]
```

1. Customer deposits funds to their internal account
2. Platform creates quote (internal -> external)
3. Execute quote - funds sent to beneficiary

**Use case:** AP automation, payroll, regular payouts

### Pattern 2: JIT Funded Payouts

```
[Customer Bank] -> [Grid Settlement Account] -> [Beneficiary External Account]
```

1. Create quote with JIT source
2. Customer receives payment instructions
3. Customer sends funds to Grid settlement account
4. Grid automatically forwards to beneficiary

**Use case:** One-off payments, lower balance requirements

### Pattern 3: Platform Rewards

```
[Platform Internal Account] -> [User External Wallet]
```

1. Platform holds USD in internal account
2. Creates quote for USD to BTC
3. Executes quote sending BTC to user's wallet

**Use case:** Bitcoin rewards, cashback, referrals

### Pattern 4: Crypto On-Ramp

```
[User Bank] -> [User Internal Account] -> [User Crypto Wallet]
```

1. User deposits fiat to internal account
2. Creates quote for fiat to crypto
3. Executes quote receiving crypto in self-custody wallet

**Use case:** Crypto exchanges, wallet apps
