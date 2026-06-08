# Lightspark Grid Payout Integration — Architecture & Implementation Spec

> **Version:** 1.0  
> **Date:** 2026-06-08  
> **Status:** Approved design — ready for implementation planning  
> **Scope:** Replace Bridge.xyz as the fiat payout provider with Lightspark Grid; keep existing Turnkey USDC + Relay signing UX  
> **Approach:** Parallel `grid` module (Approach 1) + crypto-first funding adapter (Funding Model A)  
> **Source docs:** `docs/lightspark-grid-docs/` (139 files, synced from https://docs.lightspark.com/)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Locked Design Decisions](#2-locked-design-decisions)
3. [Current Anzo System (As-Is)](#3-current-anzo-system-as-is)
4. [Lightspark Grid (Target Provider)](#4-lightspark-grid-target-provider)
5. [Bridge → Grid Conceptual Mapping](#5-bridge--grid-conceptual-mapping)
6. [System Architecture](#6-system-architecture)
7. [Database Schema](#7-database-schema)
8. [KYC & Sumsub Integration](#8-kyc--sumsub-integration)
9. [Payout Flow (Prepare / Execute / Status)](#9-payout-flow-prepare--execute--status)
10. [Crypto Funding Adapter (Relay + JIT)](#10-crypto-funding-adapter-relay--jit)
11. [External Accounts (Beneficiaries)](#11-external-accounts-beneficiaries)
12. [NestJS Module Structure](#12-nestjs-module-structure)
13. [API Contract (Anzo → Mobile)](#13-api-contract-anzo--mobile)
14. [GridProvider — API Surface](#14-gridprovider--api-surface)
15. [Webhooks](#15-webhooks)
16. [State Machines & Status Enums](#16-state-machines--status-enums)
17. [Environment Variables & Configuration](#17-environment-variables--configuration)
18. [Error Handling & Idempotency](#18-error-handling--idempotency)
19. [Migration from Bridge](#19-migration-from-bridge)
20. [Testing Strategy](#20-testing-strategy)
21. [Production Checklist](#21-production-checklist)
22. [Frontend Integration Guide](#22-frontend-integration-guide)
23. [Open Questions & Lightspark Confirmations](#23-open-questions--lightspark-confirmations)
24. [Implementation Phases](#24-implementation-phases)
25. [Reference Links](#25-reference-links)

---

## 1. Executive Summary

Anzo currently routes fiat payouts through **Bridge.xyz**: users sign a **Relay** quote to move **USDC** from their **Turnkey** wallet into a Bridge-managed Base wallet, then the backend triggers `POST /v0/transfers` to deliver fiat to a saved bank account.

We are replacing Bridge as the payout rail with **Lightspark Grid** while **preserving the mobile signing UX** (Relay steps, gasless, cross-chain USDC routing). Grid becomes the compliance + FX + bank-rail orchestrator; Anzo remains the wallet + KYC data owner.

### What changes

| Layer | Before (Bridge) | After (Grid) |
|-------|-----------------|--------------|
| Payout provider | Bridge `POST /transfers` | Grid `POST /quotes` + `POST /quotes/{id}/execute` |
| Customer record | `BridgeCustomer` | `GridCustomer` |
| Funding hub | `BridgeWallet` (Base USDC) | Grid `InternalAccount` (USD/USDC) via JIT or prefunded deposit |
| KYC submission | `PartnerSubmissionHandler` → Bridge Customers API | `GridKycHandler` → Grid `POST /customers` + Grid kyc-link |
| Partner TOS | Bridge `signed_agreement_id` | **Not required** — Grid uses own KYC |
| Webhooks | `transfer.updated` | `transaction.status_change`, `verification.status_change`, `incoming_payment.received` |
| Mobile payout API | `/api/v1/bridge/payout/*` | `/api/v1/grid/payout/*` (parallel; same shape) |

### What stays the same

- `KycProfile` + Anzo Sumsub SDK flow (ID + liveness)
- `KycVerification` per-partner step tracking
- Relay signing flow in mobile app
- `Bridge*` tables and endpoints (read-only legacy; no new Bridge customers after cutover)
- Address book, transaction history patterns

---

## 2. Locked Design Decisions

These were confirmed in brainstorming:

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| D1 | Module strategy | **Parallel `grid` module** alongside `bridge` | Isolated rollout, easy feature flag, no Bridge regression |
| D2 | Funding UX | **Keep Turnkey USDC + Relay signing** | Users already hold USDC in Turnkey; no ACH prefund required for payout |
| D3 | Grid funding backend | **JIT quote + crypto deposit instructions** (with Relay adapter) | Maps today's Relay→deposit pattern to Grid's JIT model |
| D4 | KYC data source | **Reuse `KycProfile`**; Grid kyc-link as partner step | Avoid duplicate PII collection; Grid manages its own Sumsub applicant internally |
| D5 | Mobile API shape | **Mirror Bridge payout endpoints** (`prepare` / `execute` / `status`) | Minimize mobile churn |
| D6 | Bridge sunset | **Stop new Bridge payout writes** after Grid GA; keep history | `BridgeTransaction` remains for existing users |

---

## 3. Current Anzo System (As-Is)

### 3.1 Relevant Prisma models

```
User
├── KycProfile (PII, sumsubApplicantId, sumsubGovIdDocNumber, taxId, ...)
├── KycVerification[] (per partnerId + step)
├── BridgeCustomer (legacy payout KYC)
│   ├── BridgeWallet (Base USDC hub)
│   ├── BridgeExternalAccount[] (payees)
│   └── BridgeVirtualAccount[] (on-ramp only)
├── BridgeTransaction (via userId, not FK)
├── PaytrieCustomer (Canada — separate partner)
└── Wallet (Turnkey addresses: ethereum, solana, bitcoin)
```

### 3.2 Modular KYC (`src/modules/kyc/`)

**Partner configs** (`partner-steps.config.ts`):

| Partner | Required steps |
|---------|----------------|
| `bridge` | `id_verification`, `questionnaire`, `tos` |
| `paytrie` | `id_verification` |

**Auto-submission** (`partner-submission.handler.ts`):
- On step completion → `checkAndSubmitIfReady(userId)`
- Bridge: assembles `KycProfile` + Sumsub doc images + TOS → `BridgeProvider.createCustomer`
- Paytrie: `generateShareToken(sumsubApplicantId)` → `importSumsubUser(share_token)`

**Key Sumsub fields on `KycProfile`:**

| Field | Purpose |
|-------|---------|
| `sumsubApplicantId` | Direct Sumsub applicant (Anzo-owned) |
| `sumsubInspectionId` | Document image fetch |
| `sumsubReviewAnswer` | `GREEN` / `RED` |
| `sumsubIdDocType` | `PASSPORT`, `ID_CARD`, `DRIVERS`, etc. |
| `sumsubGovIdDocNumber` | OCR doc number for passport flow |

### 3.3 Bridge seamless payout (`bridge-payout.handler.ts`)

**Prepare** (`POST /bridge/payout/prepare`):
1. Validate external account + `BridgeCustomer.status === ACTIVE`
2. Require `BridgeWallet` on Base
3. `RelayProvider.getQuote` — USDC Base → Bridge wallet address
4. Create `BridgeTransaction` with `status: PENDING_DEPOSIT`, store `relayQuote` in metadata

**Execute** (`POST /bridge/payout/execute`):
1. Client sends `transactionHash` after broadcasting Relay steps
2. `status → DEPOSIT_BROADCASTING`
3. Background: poll Relay status OR wait for Base receipt
4. `status → DEPOSIT_CONFIRMED`
5. `BridgeProvider.createTransfer` from `bridge_wallet` → external account
6. Bridge webhooks update final status

**Supported destination rails** (Bridge `PAYMENT_RAIL_MAP`):

| `accountType` | Rail | Currency |
|---------------|------|----------|
| `us` / `ach` | `ach` | USD |
| `wire` | `wire` | USD |
| `sepa` | `sepa` | EUR |
| `clabe` | `spei` | MXN |
| `pix` | `pix` | BRL |
| `gb` | `faster_payments` | GBP |

### 3.4 Existing env vars (Bridge + KYC)

```bash
# Bridge (legacy — remains for historical data)
BRIDGE_API_KEY=
BRIDGE_API_URL=https://api.bridge.xyz/v0
BRIDGE_WEBHOOK_PUBLIC_KEY=
BRIDGE_DEVELOPER_FEE_PERCENT=0.5
BRIDGE_TRANSFER_DEVELOPER_FEE=0.00

# Sumsub (Anzo-owned — continues)
SUMSUB_APP_TOKEN=
SUMSUB_SECRET_KEY=
SUMSUB_LEVEL_NAME=id-and-liveness
SUMSUB_BASE_URL=https://api.sumsub.com
SUMSUB_WEBHOOK_SECRET=

# Relay (payout funding — continues)
RELAY_API_KEY=
RELAY_BASE_URL=https://api.relay.link
BASE_RPC_URL=
```

---

## 4. Lightspark Grid (Target Provider)

### 4.1 API fundamentals

| Item | Value |
|------|-------|
| Base URL | `https://api.lightspark.com/grid/2025-10-13` |
| Auth | HTTP Basic: `GRID_CLIENT_ID` / `GRID_CLIENT_SECRET` |
| Content-Type | `application/json` |
| Idempotency | `Idempotency-Key` header on POST (quotes, execute, customers) |
| Rate limits | Sandbox: 100 req/min; Production: 1000 req/min |
| Dashboard | https://app.lightspark.com |
| OpenAPI | https://docs.lightspark.com/openapi.yaml |

### 4.2 Core entities

| Entity | Typed ID | Purpose |
|--------|----------|---------|
| Customer | `Customer:...` | Individual or business; KYC-gated |
| InternalAccount | `InternalAccount:...` | Grid-managed balance (CHECKING or EMBEDDED_WALLET) |
| ExternalAccount | `ExternalAccount:...` | Beneficiary bank/wallet/UMA destination |
| Quote | `Quote:...` | Locked rate + fees; expires in 1–5 min |
| Transaction | `Transaction:...` | Payment record after quote execution |
| Verification | `Verification:...` | KYC/KYB verification record |

### 4.3 Standard payout sequence (Grid-native)

```
1. POST /customers                          → Customer:...
2. POST /customers/{id}/kyc-link          → KYC → kycStatus: APPROVED
3. GET  /internal-accounts?customerId=... → InternalAccount:... (auto-created)
4. Fund internal account:
   - Production: ACH/wire to paymentInstructions
   - Sandbox: POST /sandbox/internal-accounts/{id}/fund
5. POST /customers/{id}/external-accounts → ExternalAccount:...
6. POST /quotes                             → Quote:... (locked rate)
7. POST /quotes/{id}/execute                → Transaction:... 
8. Webhook transaction.status_change        → COMPLETED
```

### 4.4 JIT funding (Anzo adaptation)

Grid supports **just-in-time funding**: quote response includes `paymentInstructions` so the sender funds the transfer at quote time rather than maintaining a prefunded balance.

**Anzo adaptation:**
1. Create quote with `source.sourceType: "JIT"` (or hybrid — see §10)
2. Parse `paymentInstructions` from quote response
3. If instructions include **crypto deposit** (chain + address + amount): Relay USDC from Turnkey (same as Bridge flow)
4. On `incoming_payment.received` webhook (or poll): auto-execute quote OR confirm funding then execute
5. Grid delivers fiat to external account

> **Spike required:** Confirm with Lightspark sandbox whether JIT quotes for USD→MXN/EUR/etc. return USDC-on-Base deposit instructions. Fallback documented in §10.3.

### 4.5 Supported external account types (Grid)

| `accountType` | Required fields | Currency | Rail |
|---------------|-----------------|----------|------|
| `ACH` | `accountNumber`, `routingNumber` | USD | ACH (1–3 days) |
| `WIRE` | account + wire routing | USD | Wire (same day) |
| `SEPA` | `iban` | EUR | SEPA (same day) |
| `SEPA_INSTANT` | `iban` | EUR | SEPA Instant (seconds) |
| `SPEI` | `clabe` (18-digit) | MXN | SPEI/CLABE (same day) |
| `PIX` | `pixKey`, `pixKeyType` | BRL | PIX (seconds) |
| `UPI` | `upiId` | INR | UPI (seconds) |
| `INSTAPAY` | `accountNumber`, `bankName` | PHP | InstaPay (minutes) |
| `PESONET` | `accountNumber`, `bankName` | PHP | PESONet |
| `LIGHTNING_SPARK` | `address` | BTC | Lightning |
| `UMA` | `umaAddress` | varies | UMA (seconds) |

### 4.6 Grid KYC / Sumsub (provider-managed)

Grid integrates Sumsub internally. **Grid does NOT expose:**
- Sumsub `applicantId` in API responses
- Sumsub share tokens (unlike Paytrie's `importSumsubUser`)

**Grid exposes:**
- `POST /customers/{customerId}/kyc-link` → `{ url, token, expiresAt }`
- `token` — for `@sumsub/websdk` embedded flow (Grid-managed applicant)
- Webhooks: `verification.status_change`, `customer.status_change`

**Bring Your Own KYC:** Docs mention contacting Lightspark for BYO KYC. Not publicly documented. Anzo should pursue this in parallel to avoid double KYC long-term.

### 4.7 Sandbox behavior

- Customers often **auto-KYC-approved** unless magic suffix used
- Magic values (individual `lastName` suffix):
  - `001` → PENDING
  - `002` → REJECTED
  - other → APPROVED
- Fund accounts: `POST /sandbox/internal-accounts/{id}/fund`
- Test webhooks: `POST /sandbox/webhooks/send-test`

---

## 5. Bridge → Grid Conceptual Mapping

| Bridge concept | Grid equivalent | Notes |
|----------------|---------------|-------|
| `BridgeCustomer.customerId` | `GridCustomer.gridCustomerId` | `Customer:...` |
| `BridgeCustomer.kycProfileId` | `GridCustomer.kycProfileId` | Same FK pattern |
| `BridgeWallet` | `GridInternalAccount` (USD/USDC) | No separate "wallet" entity |
| `BridgeExternalAccount` | `GridExternalAccount` | Different `accountType` enum |
| `BridgeTransaction` | `GridTransaction` | Quote-driven lifecycle |
| `POST /transfers` (fixed output) | `POST /quotes` + execute | Amount locking via `amountCurrency` |
| Bridge TOS `signed_agreement_id` | N/A | Grid kyc-link replaces |
| Persona/Sumsub → Bridge submit | `KycProfile` → `POST /customers` + kyc-link | Two Sumsub instances (Anzo + Grid) until BYO |
| `transfer.updated` webhook | `transaction.status_change` | Different payload shape |
| `bridge_wallet` source rail | `InternalAccount` or JIT source | |
| Relay USDC → Bridge wallet | Relay USDC → JIT crypto instructions | Same mobile UX |
| `PAYMENT_RAIL_MAP` | Grid `accountType` on external account | Direct mapping (§11) |
| `BridgeActivityEvent` | New: `GridActivityEvent` (optional) | For in-app activity feed |

---

## 6. System Architecture

### 6.1 Component diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Mobile App (Avvio)                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ KYC Checklist│  │ Payee Mgmt   │  │ Payout Flow  │  │ Turnkey Sign │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
└─────────┼─────────────────┼─────────────────┼─────────────────┼─────────┘
          │                 │                 │                 │
          ▼                 ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Anzo Backend (NestJS)                             │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ kyc/ (existing)                                                     │  │
│  │  KycService, SumsubKycHandler, PartnerSubmissionHandler             │  │
│  │  + checkAndSubmitGrid()  [NEW]                                      │  │
│  │  + partner-steps: grid  [NEW]                                       │  │
│  └───────────────────────────────┬────────────────────────────────────┘  │
│                                  │                                       │
│  ┌───────────────────────────────▼────────────────────────────────────┐  │
│  │ grid/ [NEW MODULE]                                                  │  │
│  │  GridController                                                     │  │
│  │  GridService (facade)                                               │  │
│  │  GridProvider (HTTP client)                                         │  │
│  │  GridKycHandler                                                     │  │
│  │  GridExternalAccountHandler                                         │  │
│  │  GridPayoutHandler  ←→ RelayProvider, RelayStatusService            │  │
│  │  GridWebhookController + GridWebhookGuard                           │  │
│  │  GridActivityHandler (optional)                                     │  │
│  └───────────────────────────────┬────────────────────────────────────┘  │
│                                  │                                       │
│  ┌───────────────────────────────▼────────────────────────────────────┐  │
│  │ relay/ (existing) — USDC routing from Turnkey wallet                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ bridge/ (legacy — read-only after cutover)                          │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    ▼                             ▼
           ┌────────────────┐           ┌────────────────┐
           │ Lightspark Grid│           │ Relay Protocol │
           │   REST API     │           │  (USDC move)   │
           └───────┬────────┘           └────────────────┘
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
  ┌─────────┐ ┌─────────┐ ┌──────────┐
  │ Sumsub  │ │  Banks  │ │ Bitcoin/ │
  │ (Grid)  │ │ (Rails) │ │ Lightning│
  └─────────┘ └─────────┘ └──────────┘
```

### 6.2 Payout data flow (Funding Model A)

```
User taps "Send $100 MXN to Maria"
        │
        ▼
POST /grid/payout/prepare
        │
        ├─ Validate GridCustomer (kycStatus=APPROVED)
        ├─ Validate GridExternalAccount
        ├─ POST /quotes (JIT source, external dest, amount lock)
        ├─ Persist GridTransaction (PENDING_FUNDING)
        ├─ Parse paymentInstructions from quote
        ├─ If crypto instruction:
        │     RelayProvider.getQuote(USDC Turnkey → deposit address)
        └─ Return { gridPayoutId, steps, quote snapshot, expiresAt }
        │
        ▼
Mobile: Turnkey signs Relay steps, broadcasts
        │
        ▼
POST /grid/payout/execute { gridPayoutId, transactionHash }
        │
        ├─ status → DEPOSIT_BROADCASTING
        ├─ Background: poll Relay / Base receipt
        ├─ status → DEPOSIT_CONFIRMED
        ├─ Wait for incoming_payment.received OR poll internal account
        ├─ POST /quotes/{id}/execute (Idempotency-Key)
        └─ status → EXECUTING → (webhook) IN_FLIGHT → COMPLETED
        │
        ▼
Push notification + transaction history update
```

### 6.3 Feature flag

```typescript
// App config or env
GRID_PAYOUT_ENABLED=true|false
PAYOUT_PROVIDER=grid|bridge  // per-user override in app-config if needed
```

New users after GA: `PAYOUT_PROVIDER=grid`. Existing Bridge-active users: grandfathered until migrated.

---

## 7. Database Schema

### 7.1 New Prisma models

Add to `prisma/schema.prisma`:

```prisma
// ============================================
// Lightspark Grid Integration Models
// ============================================

model GridCustomer {
  id                 String   @id @default(cuid())
  userId             String   @unique
  gridCustomerId     String   @unique   // Customer:0195...
  platformCustomerId String   @unique   // Anzo userId (immutable link)
  customerType       String   @default("INDIVIDUAL")
  region             String             // ISO-2 country; immutable on Grid
  kycStatus          String   @default("NOT_STARTED")
  // NOT_STARTED | PENDING | IN_REVIEW | APPROVED | REJECTED
  status             String   @default("PENDING_KYC")
  // PENDING_KYC | ACTIVE | SUSPENDED | FAILED
  kycProfileId       String?  @unique
  signedAgreementId  String?            // unused for Grid; reserved
  kycRejectionReason String?
  kycRejectionCount  Int      @default(0)
  metadata           Json?              // last kyc-link response, verification errors
  createdAt          DateTime @default(now())
  updatedAt          DateTime @updatedAt

  user               User            @relation(fields: [userId], references: [id], onDelete: Cascade)
  kycProfile         KycProfile?     @relation(fields: [kycProfileId], references: [id])
  internalAccounts   GridInternalAccount[]
  externalAccounts   GridExternalAccount[]
  transactions       GridTransaction[]

  @@index([gridCustomerId])
  @@index([platformCustomerId])
  @@index([status])
  @@index([kycStatus])
}

model GridInternalAccount {
  id                  String   @id @default(cuid())
  gridCustomerId      String
  gridAccountId       String   @unique   // InternalAccount:...
  currency            String             // USD, USDC, EUR, etc.
  accountType         String   @default("CHECKING")
  status              String   @default("ACTIVE")
  balance             String?  @default("0")
  paymentInstructions Json?              // ACH routing, reference, wire instructions
  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  gridCustomer        GridCustomer @relation(fields: [gridCustomerId], references: [id], onDelete: Cascade)

  @@index([gridCustomerId])
  @@index([gridAccountId])
  @@unique([gridCustomerId, currency])
}

model GridExternalAccount {
  id                    String   @id @default(cuid())
  gridCustomerId        String
  gridExternalAccountId String   @unique   // ExternalAccount:...
  accountType           String             // ACH, SPEI, PIX, UPI, SEPA, etc.
  currency              String
  accountOwnerName      String
  accountName           String?            // user-defined label
  last4                 String?
  metadata              Json?              // clabe, iban, pixKey, upiId, routingNumber, etc.
  isActive              Boolean  @default(true)
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  gridCustomer          GridCustomer @relation(fields: [gridCustomerId], references: [id], onDelete: Cascade)

  @@index([gridCustomerId])
  @@index([gridExternalAccountId])
}

model GridTransaction {
  id                    String    @id @default(cuid())
  userId                String
  gridCustomerId        String
  gridTransactionId     String?   @unique   // Transaction:... after execute
  gridQuoteId           String?   @unique   // Quote:...
  type                  String    @default("PAYOUT")
  status                String    @default("PENDING_QUOTE")
  amount                String              // user-facing payout amount
  sourceAmount          String?
  sourceCurrency        String?
  destinationAmount     String?
  destinationCurrency   String?
  exchangeRate          String?
  fees                  String?
  feesCurrency          String?
  depositTxHash         String?
  depositAddress        String?
  errorMessage          String?
  metadata              Json?               // quoteSnapshot, relayQuote, payoutContext, fundingInstructions
  createdAt             DateTime  @default(now())
  updatedAt             DateTime  @updatedAt
  completedAt           DateTime?

  gridCustomer          GridCustomer @relation(fields: [gridCustomerId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([gridCustomerId])
  @@index([gridQuoteId])
  @@index([gridTransactionId])
  @@index([status])
  @@index([type])
}

model GridActivityEvent {
  id         String   @id @default(cuid())
  userId     String
  externalId String   @unique
  type       String
  status     String   @default("COMPLETED")
  title      String?
  subtitle   String?
  sourceId   String?
  metadata   Json?
  occurredAt DateTime @default(now())
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([userId, occurredAt])
}
```

### 7.2 User model additions

```prisma
model User {
  // ... existing fields ...
  GridCustomer        GridCustomer?
  GridActivityEvent   GridActivityEvent[]
}
```

### 7.3 KycProfile addition

```prisma
model KycProfile {
  // ... existing fields ...
  gridCustomer        GridCustomer?
}
```

### 7.4 Tables unchanged (legacy)

| Table | Treatment |
|-------|-----------|
| `BridgeCustomer` | No new writes after cutover; read for history |
| `BridgeWallet` | Legacy |
| `BridgeExternalAccount` | Legacy; optional one-time migration script to `GridExternalAccount` |
| `BridgeTransaction` | Legacy payout history |
| `BridgeActivityEvent` | Legacy activity feed |
| `BridgeVirtualAccount` | On-ramp only; out of Grid scope v1 |

### 7.5 Migration SQL notes

Migration name suggestion: `20260608120000_add_grid_integration`

- All new tables; no destructive changes
- Backfill NOT required for existing users
- `platformCustomerId` = `User.id` (cuid)

---

## 8. KYC & Sumsub Integration

### 8.1 Dual Sumsub reality

Anzo operates **two Sumsub touchpoints**:

| Instance | Owner | When | Applicant ID |
|----------|-------|------|--------------|
| Anzo Sumsub | Anzo dashboard | `POST /kyc/sumsub/session` | Stored in `KycProfile.sumsubApplicantId` |
| Grid Sumsub | Lightspark (via Grid) | `POST /customers/{id}/kyc-link` | Internal to Grid; not exposed |

**v1 approach:** User completes Anzo Sumsub first (existing flow). When Grid partner steps are ready, user also completes Grid kyc-link **unless** Lightspark enables BYO KYC.

**v2 goal:** Negotiate BYO KYC with Lightspark — pass Anzo Sumsub verification status so Grid kyc-link is skipped.

### 8.2 New partner config

Add to `partner-steps.config.ts`:

```typescript
grid: {
  partnerId: 'grid',
  steps: [
    { step: 'id_verification', required: true },   // Anzo Sumsub (existing)
    { step: 'questionnaire', required: true },      // KycProfile (existing)
    { step: 'grid_kyc', required: true },           // Grid kyc-link completion
  ],
},
```

**Removed vs Bridge:** No `tos` step — Grid does not use Bridge TOS.

### 8.3 GridKycHandler — customer creation

**Trigger:** `PartnerSubmissionHandler.checkAndSubmitGrid(userId)` after Anzo steps `id_verification` + `questionnaire` are approved.

**Skip conditions:**
- `GridCustomer` already exists with `gridCustomerId`
- `kycStatus === APPROVED` and `status === ACTIVE`

**Mapping: KycProfile → POST /customers**

| KycProfile field | Grid field | Transform |
|------------------|------------|-----------|
| `userId` | `platformCustomerId` | direct |
| — | `customerType` | `"INDIVIDUAL"` |
| `country` | `region` | alpha-3 → alpha-2 (`USA` → `US`) |
| `firstName` + `middleName` + `lastName` | `fullName` | join with spaces |
| `dateOfBirth` | `birthDate` | `YYYY-MM-DD` |
| `nationality` or `country` | `nationality` | alpha-3 → alpha-2 |
| `user.email` | `email` | from User relation |
| `phone` | `phone` | E.164 |
| `streetLine1` | `address.line1` | |
| `streetLine2` | `address.line2` | optional |
| `city` | `address.city` | |
| `state` | `address.state` | |
| `postalCode` | `address.postalCode` | |
| `country` | `address.country` | alpha-2 |
| `taxIdType` + `taxId` | `identification` | map SSN/PASSPORT/ITIN |
| — | `currencies` | `["USD", "USDC"]` per platform config |

**Identification type mapping:**

| KycProfile `taxIdType` / country | Grid `identification.type` |
|----------------------------------|---------------------------|
| USA `ssn` | `SSN` |
| Passport flow | `PASSPORT` |
| Other | `ITIN` or omit if not US |

**Idempotency key:** `grid-customer-${userId}-${sha256(payload)}`

**On success:** Upsert `GridCustomer`, link `kycProfileId`, set `kycStatus` from response.

**Post-create:**
1. `GET /internal-accounts?customerId={gridCustomerId}` → upsert `GridInternalAccount` rows
2. If `kycStatus !== APPROVED`: generate kyc-link (§8.4)

### 8.4 Grid kyc-link flow

**Endpoint:** `POST /customers/{gridCustomerId}/kyc-link`

**Request:**
```json
{
  "redirectUrl": "https://api.avvio.xyz/api/v1/grid/kyc/callback?userId={userId}"
}
```

**Response stored in `GridCustomer.metadata.lastKycLink`:**
```json
{
  "url": "https://verify.sumsub.com/...",
  "token": "eyJ...",
  "expiresAt": "2026-06-09T10:00:00Z"
}
```

**Anzo API for mobile:**

| Method | Endpoint | Response |
|--------|----------|----------|
| GET | `/api/v1/kyc/status/grid` | steps including `grid_kyc` |
| POST | `/api/v1/grid/kyc/link` | `{ url, token, expiresAt }` |

**Callback** (`GET /api/v1/grid/kyc/callback`):
- Reads `userId` query param
- Returns HTML with deep link `anzo://kyc/grid-complete`
- Does NOT mark step approved — webhook is source of truth

**Mobile options:**
1. **Redirect** to `url` (recommended for v1)
2. **Embedded** `@sumsub/websdk` with Grid `token` (not Anzo Sumsub token)

### 8.5 KYC webhook handling

On `verification.status_change`:

```typescript
async handleVerificationStatusChange(data: {
  verificationId: string;
  customerId: string;      // Customer:...
  status: string;          // APPROVED | REJECTED | RESOLVE_ERRORS | IN_REVIEW
  previousStatus: string;
  kycStatus: string;
  errors?: Array<{ type: string; field?: string; reason: string }>;
  completedAt?: string;
}) {
  // 1. Find GridCustomer by gridCustomerId = data.customerId
  // 2. Update kycStatus, status
  // 3. If APPROVED:
  //    - Mark KycVerification (partnerId=grid, step=grid_kyc) as approved
  //    - Set GridCustomer.status = ACTIVE
  //    - Refresh internal accounts
  //    - Push notification
  // 4. If REJECTED:
  //    - Mark grid_kyc step rejected with errors
  //    - Increment kycRejectionCount
  // 5. If RESOLVE_ERRORS:
  //    - Mark grid_kyc as in_progress; surface errors to mobile
}
```

On `customer.status_change`:
- Sync `kycStatus` if `changedFields` includes kyc fields
- Update `GridCustomer.status`

### 8.6 Sumsub applicant ID — what we do NOT pass

| Mechanism | Bridge | Paytrie | Grid |
|-----------|--------|---------|------|
| Direct applicant ID | via Persona extraction | — | **Not supported** |
| Share token | — | `generateShareToken` → `importSumsubUser` | **Not supported** |
| Document images base64 | Bridge `identifying_information` | — | Grid kyc-link handles |
| PII from questionnaire | `createCustomer` payload | registration payload | `POST /customers` payload |

`KycProfile.sumsubApplicantId` remains for Anzo/Paytrie only. Audit log only for Grid.

### 8.7 KYC status gating for payouts

Payout `prepare` requires:
```typescript
gridCustomer.kycStatus === 'APPROVED'
gridCustomer.status === 'ACTIVE'
```

Error code: `KYC_NOT_APPROVED` (maps from Grid API).

---

## 9. Payout Flow (Prepare / Execute / Status)

### 9.1 Anzo-layer status machine

```
PENDING_QUOTE        Quote creation failed or not started
       ↓
QUOTED               Grid quote created; Relay steps returned
       ↓
PENDING_FUNDING      Alias for QUOTED when awaiting user sign
       ↓
DEPOSIT_BROADCASTING User signed; txHash recorded
       ↓
DEPOSIT_CONFIRMED    Relay/Base receipt confirmed; Grid funding detected
       ↓
EXECUTING            POST /quotes/{id}/execute called
       ↓
IN_FLIGHT            Grid transaction IN_FLIGHT (from webhook)
       ↓
COMPLETED            Grid transaction COMPLETED

Terminal: FAILED | QUOTE_EXPIRED | CANCELLED | RETURNED
```

### 9.2 Prepare — `GridPayoutHandler.preparePayout`

**Input DTO** (mirror Bridge):
```typescript
class PrepareGridPayoutDto {
  externalAccountId: string;  // GridExternalAccount.id (internal DB)
  amount: string;               // decimal string, min $1.00
  currency: string;             // USD | EUR | MXN | BRL | GBP | INR
  amountLock?: 'SENDING' | 'RECEIVING';  // default RECEIVING for fixed-output FX
}
```

**Steps:**

1. **Validate user**
   - `GridCustomer` exists, `kycStatus=APPROVED`, `status=ACTIVE`
   - `GridExternalAccount` belongs to user, `isActive=true`

2. **Resolve Grid accounts**
   - Source: JIT quote (no prefunded balance required) OR USDC `InternalAccount` if fallback
   - Destination: `GridExternalAccount.gridExternalAccountId`

3. **Create Grid quote**
```json
POST /quotes
{
  "source": {
    "sourceType": "JIT",
    "customerId": "Customer:..."
  },
  "destination": {
    "destinationType": "ACCOUNT",
    "accountId": "ExternalAccount:..."
  },
  "amount": "100.00",
  "amountCurrency": "MXN",
  "amountLock": "RECEIVING"
}
```

> Adjust `amountLock` / `amountCurrency` per corridor. USD ACH may use `SENDING` with `amountCurrency: USD`.

4. **Persist `GridTransaction`**
```typescript
{
  status: 'QUOTED',
  gridQuoteId: quote.id,
  amount: dto.amount,
  destinationCurrency: dto.currency,
  sourceAmount: quote.sourceAmount,
  destinationAmount: quote.destinationAmount,
  exchangeRate: quote.exchangeRate,
  fees: quote.fees,
  metadata: {
    quoteSnapshot: quote,
    payoutContext: {
      externalAccountId: dto.externalAccountId,
      gridExternalAccountId: ext.gridExternalAccountId,
      gridCustomerId: customer.gridCustomerId,
      expiresAt: quote.expiresAt,
    },
    fundingInstructions: quote.paymentInstructions,
  },
}
```

5. **Build Relay quote** (if crypto funding instructions present)
   - Parse deposit address + amount + chain from `paymentInstructions`
   - Default chain: Base (8453), token: USDC `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
   - `RelayProvider.getQuote({ user: wallet.ethereumAddress, recipient: depositAddress, ... })`
   - Store `relayQuote` in metadata

6. **Return response** (Bridge-compatible shape)
```json
{
  "gridPayoutId": "clxyz...",
  "quoteId": "Quote:...",
  "steps": [ "...relay steps..." ],
  "gasless": true,
  "depositAddress": "0x...",
  "amount": "99.98",
  "requestedAmount": "100.00",
  "amountAdjusted": false,
  "currency": "MXN",
  "externalAccountId": "clxyz...",
  "exchangeRate": "17.05",
  "fees": "7.50",
  "feesCurrency": "USD",
  "expiresAt": "2026-06-08T12:05:00.000Z"
}
```

### 9.3 Execute — `GridPayoutHandler.executePayout`

**Input DTO:**
```typescript
class ExecuteGridPayoutDto {
  gridPayoutId: string;
  transactionHash: string;       // required — EVM tx hash on Base
  requestId?: string;            // Relay request id if available
}
```

**Steps:**

1. Load `GridTransaction` where `id=gridPayoutId`, `userId`, `status in [QUOTED, PENDING_FUNDING]`
2. `status → DEPOSIT_BROADCASTING`; store `depositTxHash`
3. Background `pollAndExecuteQuote(transactionId)`:
   - Poll Relay status OR `waitForBaseTransactionSuccess(txHash)`
   - `status → DEPOSIT_CONFIRMED`
   - Wait for Grid funding confirmation:
     - **Primary:** `incoming_payment.received` webhook sets `fundingConfirmed: true`
     - **Fallback:** Poll `GET /quotes/{id}` or `GET /internal-accounts/{id}` balance (timeout 5 min)
   - Check quote not expired; if expired → `QUOTE_EXPIRED`, require re-prepare
   - `POST /quotes/{gridQuoteId}/execute` with `Idempotency-Key: grid-exec-{gridPayoutId}`
   - Store `gridTransactionId`, `status → EXECUTING`
4. Return immediately:
```json
{
  "gridPayoutId": "clxyz...",
  "status": "DEPOSIT_BROADCASTING",
  "gasless": true,
  "transactionHash": "0x...",
  "message": "Payout submitted. Funds will be transferred to your bank account automatically."
}
```

### 9.4 Status — `GridPayoutHandler.getPayoutStatus`

**Response:**
```json
{
  "gridPayoutId": "clxyz...",
  "status": "IN_FLIGHT",
  "amount": "100.00",
  "currency": "mxn",
  "depositTxHash": "0x...",
  "gridQuoteId": "Quote:...",
  "gridTransactionId": "Transaction:...",
  "destinationAmount": "1705.00",
  "destinationCurrency": "mxn",
  "exchangeRate": "17.05",
  "fees": "7.50",
  "errorMessage": null,
  "createdAt": "...",
  "completedAt": null
}
```

### 9.5 Quote preview — `getPayoutQuote`

Lightweight fee math endpoint (no Grid/Relay calls) — mirror `BridgePayoutHandler.getPayoutQuote`:
- Reads `GRID_TRANSFER_DEVELOPER_FEE` flat USD fee
- Returns `minimumAmount: 1.00`
- Note: real FX/fees come from `POST /quotes` at prepare time

---

## 10. Crypto Funding Adapter (Relay + JIT)

### 10.1 Problem statement

Grid's documented happy path assumes **prefunded** `InternalAccount` (ACH deposit). Anzo users hold **USDC in Turnkey** and sign **Relay** operations. We must bridge these models without changing mobile UX.

### 10.2 Primary path — JIT + crypto paymentInstructions

```
POST /quotes { sourceType: "JIT", ... }
        ↓
quote.paymentInstructions contains:
  - cryptoDepositAddress (Base USDC)
  - cryptoAmount
  - reference / memo (if required)
        ↓
Relay: Turnkey USDC → cryptoDepositAddress
        ↓
Grid detects deposit → incoming_payment.received
        ↓
POST /quotes/{id}/execute
```

**Implementation in `GridPayoutHandler.parseFundingInstructions(quote)`:**

```typescript
interface CryptoFundingInstruction {
  chainId: number;           // 8453 Base default
  tokenAddress: string;      // USDC Base
  depositAddress: string;
  amountAtomic: string;      // smallest unit
  amountDecimal: string;
  reference?: string;
  expiresAt: string;
}

function parseFundingInstructions(quote: GridQuoteResponse): CryptoFundingInstruction | null {
  const pi = quote.paymentInstructions;
  if (!pi) return null;
  // Parse per Lightspark OpenAPI schema — field names TBD in spike
  // Candidates: pi.cryptoAddress, pi.evmAddress, pi.stablecoin, pi.amount, pi.currency
  ...
}
```

### 10.3 Fallback path A — Prefund USDC internal account via Relay

If JIT does not return crypto instructions:

1. `GET /internal-accounts?customerId=...&currency=USDC`
2. Read `paymentInstructions` for USDC account (deposit address)
3. Relay USDC from Turnkey → Grid USDC internal account
4. On `incoming_payment.received`: create **new** quote with `sourceType: "ACCOUNT"`, `accountId: InternalAccount:usdc-...`
5. Execute immediately (balance already funded)

### 10.4 Fallback path B — Ramp quote USDC→USD

Use `08-ramps/` flow:
1. Quote USDC internal → USD internal (or direct to payout)
2. Chain with payout quote

Only if paths 10.2/10.3 fail in sandbox spike.

### 10.5 Fallback path C — Sandbox fund + ACCOUNT source

**Sandbox only:**
```bash
POST /sandbox/internal-accounts/{id}/fund
{ "amount": "10000.00", "currency": "USD" }
```
Then `sourceType: "ACCOUNT"` — for E2E tests without Relay.

### 10.6 Relay integration (unchanged from Bridge)

Reuse from `bridge-payout.handler.ts`:
- `RelayProvider.getQuote`
- `RelayStatusService.pollUntilResolved`
- `waitForBaseTransactionSuccess`
- `relayProvider.indexTransaction(8453, txHash)`
- `resolveRelayRequestId`

**Constants:**
```typescript
const BASE_CHAIN_ID = 8453;
const USDC_DECIMALS = 6;
const USDC_BASE = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const MIN_PAYOUT_USD = 1.0;
```

### 10.7 Quote expiration handling

Grid quotes expire in **1–5 minutes**.

| Scenario | Action |
|----------|--------|
| User slow to sign | Return `expiresAt` to mobile; warn at 60s remaining |
| Quote expired before execute | `status → QUOTE_EXPIRED`; client must re-prepare |
| Deposit confirmed but quote expired | Re-create quote with same params; match rate change UX |

### 10.8 Amount locking strategies

| Corridor | `amountLock` | `amountCurrency` | User sees |
|----------|--------------|------------------|-----------|
| USD ACH | `RECEIVING` or `SENDING` | `USD` | 1:1 USDC→USD |
| MXN SPEI | `RECEIVING` | `MXN` | Fixed peso amount |
| EUR SEPA | `RECEIVING` | `EUR` | Fixed euro amount |
| BRL PIX | `RECEIVING` | `BRL` | Fixed real amount |
| GBP | `RECEIVING` | `GBP` | Fixed pound amount |
| INR UPI | `RECEIVING` | `INR` | Fixed rupee amount |

---

## 11. External Accounts (Beneficiaries)

### 11.1 Bridge → Grid account type mapper

File: `src/modules/grid/utils/grid-account-type.mapper.ts`

| Anzo/Bridge `accountType` | Grid `accountType` | Grid payload fields |
|---------------------------|-------------------|---------------------|
| `us` | `ACH` | `accountNumber`, `routingNumber` |
| `ach` | `ACH` | `accountNumber`, `routingNumber` |
| `wire` | `WIRE` | `accountNumber`, `routingNumber` (wire) |
| `sepa` | `SEPA` or `SEPA_INSTANT` | `iban`, optional `bic` |
| `clabe` | `SPEI` | `clabe` (18-digit) |
| `pix` | `PIX` | `pixKey`, `pixKeyType`, `documentNumber` |
| `gb` | (Faster Payments — confirm enum with OpenAPI) | `sortCode`, `accountNumber` |
| *(new)* | `UPI` | `upiId` |

**PIX key types:** `EMAIL`, `PHONE`, `CPF`, `CNPJ`, `RANDOM`

### 11.2 Create external account — Anzo API

`POST /api/v1/grid/external-accounts`

Reuse Bridge DTO structure from `create-external-account.dto.ts` with Grid-specific validation. Request body stays flat (no wrapper) for mobile compatibility.

**Example — Mexican CLABE:**
```json
{
  "currency": "mxn",
  "accountType": "clabe",
  "bankName": "BBVA Mexico",
  "accountName": "Maria's Account",
  "accountOwnerType": "individual",
  "accountOwnerName": "Maria Garcia",
  "firstName": "Maria",
  "lastName": "Garcia",
  "clabeAccount": { "accountNumber": "626899715090851234" },
  "address": {
    "streetLine1": "Av. Reforma 123",
    "city": "Mexico City",
    "state": "CDMX",
    "postalCode": "06600",
    "country": "MEX"
  }
}
```

**Grid API call:**
```json
POST /customers/{gridCustomerId}/external-accounts
{
  "accountType": "SPEI",
  "clabe": "626899715090851234",
  "currency": "MXN",
  "accountOwnerName": "Maria Garcia"
}
```

**Persist:**
```typescript
await prisma.gridExternalAccount.create({
  data: {
    gridCustomerId: customer.id,
    gridExternalAccountId: resp.id,
    accountType: 'SPEI',
    currency: 'mxn',
    accountOwnerName: 'Maria Garcia',
    last4: clabe.slice(-4),
    metadata: { clabe, bankName, address, ... },
  },
});
```

### 11.3 List / delete

| Method | Anzo endpoint | Grid API |
|--------|---------------|----------|
| GET | `/grid/external-accounts` | `GET /customers/{id}/external-accounts` |
| GET | `/grid/external-accounts/:id` | `GET /customers/{id}/external-accounts/{extId}` |
| DELETE | `/grid/external-accounts/:id` | `DELETE /customers/{id}/external-accounts/{extId}` |

List response mirrors Bridge shape for mobile:
```json
[{
  "id": "clxyz...",
  "gridExternalAccountId": "ExternalAccount:...",
  "bankName": "BBVA Mexico",
  "accountName": "Maria's Account",
  "last4": "1234",
  "accountType": "clabe",
  "currency": "mxn",
  "accountOwnerName": "Maria Garcia"
}]
```

### 11.4 Optional: migrate Bridge payees

One-time script for existing users:
```
BridgeExternalAccount → GridExternalAccount
```
- Only if same user has both `BridgeCustomer` and `GridCustomer`
- Re-create on Grid API (IDs differ); do not copy Bridge external account IDs
- Mark `metadata.migratedFromBridgeId`

### 11.5 Address book integration

`AddressBookContact` stores wallet addresses (EVM/SOL/BTC). Grid external accounts are **bank rails**, not wallet addresses. No direct merge in v1.

Future: link `AddressBookContact` to `GridExternalAccount` via `metadata.addressBookContactId`.

---

## 12. NestJS Module Structure

### 12.1 Directory layout

```
src/modules/grid/
├── grid.module.ts
├── grid.controller.ts
├── grid.service.ts                    # Facade
├── index.ts
├── providers/
│   └── grid.provider.ts               # Axios HTTP client
├── handlers/
│   ├── grid-kyc.handler.ts
│   ├── grid-external-account.handler.ts
│   ├── grid-payout.handler.ts
│   ├── grid-activity.handler.ts       # optional — in-app feed
│   └── grid-webhook.handler.ts        # event routing logic
├── webhooks/
│   ├── grid-webhook.controller.ts
│   └── grid-webhook.types.ts
├── guards/
│   └── grid-webhook.guard.ts
├── dto/
│   ├── create-grid-external-account.dto.ts
│   ├── prepare-grid-payout.dto.ts
│   ├── execute-grid-payout.dto.ts
│   ├── grid-payout-quote.dto.ts
│   └── index.ts
└── utils/
    ├── grid-account-type.mapper.ts
    ├── grid-kyc-payload.mapper.ts     # KycProfile → Grid customer
    ├── grid-error.util.ts
    ├── grid-webhook-verify.util.ts
    └── grid-country.util.ts           # alpha-3 ↔ alpha-2
```

### 12.2 grid.module.ts dependencies

```typescript
@Module({
  imports: [
    DatabaseModule,
    ConfigModule,
    RelayModule,        // existing — RelayProvider, RelayStatusService
    KycModule,          // forwardRef if needed for PartnerSubmissionHandler
    NotificationsModule // push on payout complete
  ],
  controllers: [GridController, GridWebhookController],
  providers: [
    GridService,
    GridProvider,
    GridKycHandler,
    GridExternalAccountHandler,
    GridPayoutHandler,
    GridWebhookHandler,
    GridActivityHandler,
    GridWebhookGuard,
  ],
  exports: [GridProvider, GridKycHandler],
})
export class GridModule {}
```

### 12.3 app.module.ts

```typescript
import { GridModule } from './modules/grid/grid.module';

@Module({
  imports: [
    // ...existing...
    GridModule,
  ],
})
```

### 12.4 kyc.module.ts changes

- Import `GridModule` (forwardRef) into `PartnerSubmissionHandler`
- Add `checkAndSubmitGrid()` method
- Export nothing new

### 12.5 partner-submission.handler.ts changes

```typescript
async checkAndSubmitIfReady(userId: string): Promise<void> {
  // ...existing...
  await this.checkAndSubmitBridge(userId, profile);
  await this.checkAndSubmitPaytrie(userId, profile);
  await this.checkAndSubmitGrid(userId, profile);  // NEW
}

private async checkAndSubmitGrid(userId: string, profile: any): Promise<void> {
  if (!this.gridProvider.isEnabled()) return;
  // Require: id_verification + questionnaire approved for partnerId=grid
  // Delegate to GridKycHandler.createOrUpdateCustomer(userId, profile)
}
```

### 12.6 Transaction history integration

`transaction-history.handler.ts` — add Grid payout rows:
- Query `GridTransaction` where `userId` and terminal statuses
- Map to unified history item:
```typescript
{
  id: gridTx.id,
  type: 'PAYOUT',
  service: 'grid',
  status: mapGridStatus(gridTx.status),
  fromAsset: 'USDC',
  toAsset: gridTx.destinationCurrency,
  fromAmount: gridTx.sourceAmount,
  toAmount: gridTx.destinationAmount,
  transactionHash: gridTx.depositTxHash,
  createdAt: gridTx.createdAt,
}
```

---

## 13. API Contract (Anzo → Mobile)

Base path: `/api/v1/grid`  
Auth: `Authorization: Bearer <JWT>` (same as Bridge)

### 13.1 KYC endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/kyc/status` | Grid customer KYC state + checklist steps |
| POST | `/kyc/link` | Generate Grid kyc-link `{ url, token, expiresAt }` |
| GET | `/kyc/callback` | WebView redirect handler (no auth — signed token or userId param) |

`GET /kyc/status` response:
```json
{
  "partnerId": "grid",
  "gridCustomerId": "Customer:...",
  "kycStatus": "APPROVED",
  "status": "ACTIVE",
  "paymentsEnabled": true,
  "steps": [
    { "step": "id_verification", "status": "approved", "required": true },
    { "step": "questionnaire", "status": "approved", "required": true },
    { "step": "grid_kyc", "status": "approved", "required": true }
  ]
}
```

### 13.2 External account endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/external-accounts` | Add payee |
| GET | `/external-accounts` | List payees |
| DELETE | `/external-accounts/:id` | Soft-delete (isActive=false) + Grid DELETE |

### 13.3 Payout endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/payout/quote` | Fee preview (local math only) |
| POST | `/payout/prepare` | Create Grid quote + Relay steps |
| POST | `/payout/execute` | Record txHash, trigger background execute |
| GET | `/payout/:gridPayoutId/status` | Poll status |

### 13.4 Webhook endpoint

| Method | Path | Auth |
|--------|------|------|
| POST | `/webhook` | HMAC `X-Grid-Signature` + `X-Grid-Timestamp` |

Registered at platform level:
```
https://api.avvio.xyz/api/v1/grid/webhook
```

### 13.5 Error responses

| HTTP | Code | When |
|------|------|------|
| 400 | `KYC_NOT_APPROVED` | Grid customer not approved |
| 400 | `QUOTE_EXPIRED` | Quote timed out |
| 400 | `INSUFFICIENT_FUNDS` | Relay/Grid balance error |
| 400 | `UNSUPPORTED_CORRIDOR` | Currency/rail not enabled |
| 400 | `AMOUNT_TOO_SMALL` | Below $1 minimum |
| 404 | `PAYEE_NOT_FOUND` | External account missing |
| 404 | `PAYOUT_NOT_FOUND` | Invalid gridPayoutId |
| 409 | `PAYOUT_ALREADY_EXECUTED` | Duplicate execute |
| 503 | `GRID_DISABLED` | `GRID_ENABLED=false` |

---

## 14. GridProvider — API Surface

File: `src/modules/grid/providers/grid.provider.ts`

Pattern: mirror `bridge.provider.ts` — Axios instance, Basic auth interceptor, structured logging, error formatting.

```typescript
@Injectable()
export class GridProvider {
  isEnabled(): boolean;

  // Platform
  getPlatform(): Promise<GridPlatform>;
  updatePlatform(payload: Partial<GridPlatform>): Promise<GridPlatform>;

  // Customers
  createCustomer(payload: GridCreateCustomerPayload, idempotencyKey?: string): Promise<GridCustomer>;
  getCustomer(customerId: string): Promise<GridCustomer>;
  updateCustomer(customerId: string, payload: Partial<GridCreateCustomerPayload>): Promise<GridCustomer>;
  listCustomers(params?: GridListCustomersParams): Promise<{ data: GridCustomer[] }>;
  createKycLink(customerId: string, redirectUrl?: string): Promise<GridKycLink>;

  // Internal accounts
  listInternalAccounts(params: { customerId?: string; currency?: string }): Promise<{ data: GridInternalAccount[] }>;
  getInternalAccount(accountId: string): Promise<GridInternalAccount>;

  // External accounts
  createExternalAccount(customerId: string, payload: GridExternalAccountPayload): Promise<GridExternalAccount>;
  listExternalAccounts(customerId: string): Promise<{ data: GridExternalAccount[] }>;
  getExternalAccount(customerId: string, accountId: string): Promise<GridExternalAccount>;
  deleteExternalAccount(customerId: string, accountId: string): Promise<void>;

  // Quotes & transactions
  createQuote(payload: GridCreateQuotePayload, idempotencyKey?: string): Promise<GridQuote>;
  getQuote(quoteId: string): Promise<GridQuote>;
  executeQuote(quoteId: string, idempotencyKey?: string): Promise<GridTransaction>;
  getTransaction(transactionId: string): Promise<GridTransaction>;
  listTransactions(params?: GridListTransactionsParams): Promise<{ data: GridTransaction[] }>;

  // Same-currency (optional v1)
  createTransferOut(payload: GridTransferOutPayload): Promise<GridTransaction>;

  // Sandbox helpers
  fundInternalAccount(accountId: string, amount: string, currency: string): Promise<void>;
  sendTestWebhook(eventType: string): Promise<void>;

  // Verifications
  getVerification(verificationId: string): Promise<GridVerification>;
  listVerifications(params?: { customerId?: string; status?: string }): Promise<{ data: GridVerification[] }>;
}
```

**Auth setup:**
```typescript
this.client = axios.create({
  baseURL: config.get('GRID_BASE_URL'),
  timeout: 30000,
  auth: {
    username: config.get('GRID_CLIENT_ID'),
    password: config.get('GRID_CLIENT_SECRET'),
  },
  headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
});
```

---

## 15. Webhooks

### 15.1 Verification algorithm

```typescript
function verifyGridWebhook(
  rawBody: Buffer,
  signatureHeader: string,  // X-Grid-Signature: sha256=<hex> OR raw hex
  timestampHeader: string,  // X-Grid-Timestamp: unix seconds
  secret: string,
): boolean {
  const timestamp = parseInt(timestampHeader, 10);
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - timestamp) > 300) return false;

  const signature = signatureHeader.replace(/^sha256=/, '');
  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${rawBody.toString('utf8')}`)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expected, 'hex'),
  );
}
```

**NestJS:** Use `rawBody` middleware for `/api/v1/grid/webhook` (same pattern as Sumsub/Bridge webhooks).

### 15.2 Event handlers

| eventType | Handler | DB updates |
|-----------|---------|------------|
| `test.webhook` | Log only | — |
| `customer.status_change` | `handleCustomerStatusChange` | `GridCustomer.kycStatus`, `status` |
| `verification.status_change` | `handleVerificationStatusChange` | `GridCustomer`, `KycVerification` step `grid_kyc` |
| `transaction.status_change` | `handleTransactionStatusChange` | `GridTransaction.status`, `completedAt` |
| `incoming_payment.received` | `handleIncomingPayment` | `GridInternalAccount.balance`, trigger pending execute |
| `internal_account.status_change` | `handleInternalAccountStatusChange` | `GridInternalAccount.balance`, `status` |
| `bulk_upload.status_change` | No-op v1 | — |

### 15.3 transaction.status_change — payout completion

```typescript
async handleTransactionStatusChange(data: {
  transactionId: string;
  status: string;           // PENDING | IN_FLIGHT | COMPLETED | FAILED | CANCELLED
  previousStatus: string;
  quoteId?: string;
  customerId?: string;
  amount?: string;
  currency?: string;
}) {
  const gridTx = await prisma.gridTransaction.findFirst({
    where: {
      OR: [
        { gridTransactionId: data.transactionId },
        { gridQuoteId: data.quoteId },
      ],
    },
  });
  if (!gridTx) return;

  const statusMap: Record<string, string> = {
    PENDING: 'EXECUTING',
    IN_FLIGHT: 'IN_FLIGHT',
    COMPLETED: 'COMPLETED',
    FAILED: 'FAILED',
    CANCELLED: 'CANCELLED',
  };

  await prisma.gridTransaction.update({
    where: { id: gridTx.id },
    data: {
      status: statusMap[data.status] || data.status,
      gridTransactionId: data.transactionId,
      completedAt: data.status === 'COMPLETED' ? new Date() : undefined,
    },
  });

  if (data.status === 'COMPLETED') {
    await notifications.sendPayoutComplete(gridTx.userId, gridTx);
    await gridActivityHandler.recordPayoutCompleted(gridTx);
  }
}
```

### 15.4 incoming_payment.received — funding trigger

```typescript
async handleIncomingPayment(data: {
  accountId: string;
  customerId: string;
  amount: string;
  currency: string;
  transactionId?: string;
}) {
  // Update internal account balance
  await prisma.gridInternalAccount.updateMany({
    where: { gridAccountId: data.accountId },
    data: { balance: data.amount },
  });

  // Find pending payouts awaiting funding for this customer
  const pending = await prisma.gridTransaction.findMany({
    where: {
      gridCustomer: { gridCustomerId: data.customerId },
      status: 'DEPOSIT_CONFIRMED',
      metadata: { path: ['fundingConfirmed'], equals: false },
    },
  });

  for (const tx of pending) {
    await gridPayoutHandler.tryExecuteQuote(tx.id);
  }
}
```

### 15.5 Idempotent webhook processing

Store processed event IDs:
```typescript
// metadata.webhookEvents: string[] or separate WebhookEvent table
const eventKey = `${eventType}:${data.transactionId || data.verificationId}:${data.status}`;
if (alreadyProcessed(eventKey)) return;
```

### 15.6 Retry policy (Grid → Anzo)

Grid retries 5× with exponential backoff (1s, 2s, 4s, 8s, 16s). Handler must:
- Return `200` within 5 seconds
- Process async via `setImmediate` / queue
- Be idempotent

---

## 16. State Machines & Status Enums

### 16.1 GridTransaction (Anzo layer)

| Status | Terminal? | User-facing label |
|--------|-----------|-------------------|
| `PENDING_QUOTE` | No | Preparing... |
| `QUOTED` | No | Confirm payout |
| `PENDING_FUNDING` | No | Sign to fund |
| `DEPOSIT_BROADCASTING` | No | Processing deposit... |
| `DEPOSIT_CONFIRMED` | No | Deposit confirmed |
| `EXECUTING` | No | Sending to bank... |
| `IN_FLIGHT` | No | Payment in progress |
| `COMPLETED` | Yes | Complete |
| `FAILED` | Yes | Failed |
| `QUOTE_EXPIRED` | Yes | Quote expired — retry |
| `CANCELLED` | Yes | Cancelled |

### 16.2 Grid API transaction statuses

```
PENDING → IN_FLIGHT → COMPLETED
    ↓         ↓
  FAILED   CANCELLED
```

### 16.3 Grid quote statuses

```
ACTIVE → EXECUTED → COMPLETED
  ↓
EXPIRED
```

### 16.4 Grid customer KYC

```
NOT_STARTED → PENDING → IN_REVIEW → APPROVED
                              ↘ REJECTED
```

### 16.5 KycVerification step `grid_kyc`

```
not_started → in_progress → approved
                         ↘ rejected
```

---

## 17. Environment Variables & Configuration

### 17.1 New variables

```bash
# Lightspark Grid
GRID_ENABLED=false
GRID_BASE_URL=https://api.lightspark.com/grid/2025-10-13
GRID_CLIENT_ID=
GRID_CLIENT_SECRET=
GRID_WEBHOOK_SECRET=
GRID_TRANSFER_DEVELOPER_FEE=0.00
```

### 17.2 validation.schema.ts additions

```typescript
GRID_ENABLED: Joi.string().valid('true', 'false').optional().default('false'),
GRID_BASE_URL: Joi.string().uri().optional()
  .default('https://api.lightspark.com/grid/2025-10-13'),
GRID_CLIENT_ID: Joi.string().optional(),
GRID_CLIENT_SECRET: Joi.string().optional(),
GRID_WEBHOOK_SECRET: Joi.string().optional(),
GRID_TRANSFER_DEVELOPER_FEE: Joi.string().optional().default('0.00'),
```

When `GRID_ENABLED=true` in production, require `GRID_CLIENT_ID`, `GRID_CLIENT_SECRET`, `GRID_WEBHOOK_SECRET`.

### 17.3 .env.example additions

```bash
# Lightspark Grid (fiat payouts — replaces Bridge for new users)
# GRID_ENABLED=true
# GRID_BASE_URL=https://api.lightspark.com/grid/2025-10-13
# GRID_CLIENT_ID=
# GRID_CLIENT_SECRET=
# GRID_WEBHOOK_SECRET=
# GRID_TRANSFER_DEVELOPER_FEE=0.00
```

### 17.4 Platform configuration (one-time setup)

```bash
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "supportedCurrencies": ["USD", "USDC", "EUR", "MXN", "BRL", "GBP", "INR"],
    "webhookUrl": "https://api.avvio.xyz/api/v1/grid/webhook",
    "name": "Avvio"
  }'
```

### 17.5 AWS secrets / setup script

Add to `scripts/setup-aws-secrets.sh`:
- `GRID_CLIENT_ID`
- `GRID_CLIENT_SECRET`
- `GRID_WEBHOOK_SECRET`

---

## 18. Error Handling & Idempotency

### 18.1 Grid API error codes (payout-relevant)

| Grid error | HTTP | Anzo mapping | User message |
|------------|------|--------------|--------------|
| `QUOTE_EXPIRED` | 400 | `QUOTE_EXPIRED` | Rate expired. Please try again. |
| `INSUFFICIENT_FUNDS` | 400 | `INSUFFICIENT_FUNDS` | Insufficient balance. |
| `INVALID_ACCOUNT` | 400 | `INVALID_ACCOUNT` | Bank account invalid. |
| `UNSUPPORTED_CORRIDOR` | 400 | `UNSUPPORTED_CORRIDOR` | Currency not supported. |
| `AMOUNT_TOO_SMALL` | 400 | `AMOUNT_TOO_SMALL` | Minimum payout is $1. |
| `AMOUNT_TOO_LARGE` | 400 | `AMOUNT_TOO_LARGE` | Amount exceeds limit. |
| `BANK_REJECTED` | 400 | `BANK_REJECTED` | Bank rejected payment. |
| `KYC_NOT_APPROVED` | 400 | `KYC_NOT_APPROVED` | Complete verification first. |
| `COMPLIANCE_HOLD` | 400 | `COMPLIANCE_HOLD` | Contact support. |
| `LIMIT_EXCEEDED` | 400 | `LIMIT_EXCEEDED` | Daily limit exceeded. |
| `VALIDATION_ERROR` | 400 | `VALIDATION_ERROR` | Invalid input. |
| `CONFLICT` | 409 | `CONFLICT` | Duplicate request. |

### 18.2 Idempotency keys

| Operation | Key format |
|-----------|------------|
| Create customer | `grid-customer-{userId}-{sha256(payload)}` |
| Create quote | `grid-quote-{gridPayoutId}` |
| Execute quote | `grid-exec-{gridPayoutId}` |
| Create external account | `grid-ext-{userId}-{sha256(accountNumber+type)}` |
| Webhook processing | `{eventType}:{entityId}:{status}` |

### 18.3 Retry strategy

| Operation | Retries | Backoff |
|-----------|---------|---------|
| Grid API read (GET) | 3 | 1s, 2s, 4s |
| Grid API write (POST) | 0 (use idempotency) | — |
| Relay poll | existing `RelayStatusService` | — |
| Background execute | 1 retry on transient 5xx | 30s |

### 18.4 Logging

Log fields (never log secrets):
- `gridCustomerId`, `gridQuoteId`, `gridTransactionId`
- `userId`, `gridPayoutId`
- `corridor` (e.g. `USDC→MXN`)
- `relayRequestId`, `depositTxHash`
- Grid `X-Request-ID` response header

---

## 19. Migration from Bridge

### 19.1 Principles

1. **No big-bang cutover** — Grid runs parallel; feature flag controls new users
2. **Bridge tables are append-only legacy** after GA — no deletes
3. **Existing Bridge-active users** keep Bridge until manually migrated or auto-migrated per corridor
4. **KYC is NOT transferable** — Grid requires its own kyc-link (until BYO approved)
5. **External accounts must be re-created** on Grid (different IDs)

### 19.2 User segmentation

| Segment | Payout provider | KYC path |
|---------|-----------------|----------|
| New users (post-GA) | Grid | Anzo Sumsub + questionnaire + grid_kyc |
| Bridge ACTIVE (legacy) | Bridge | Existing Bridge/Persona/Sumsub flow |
| Bridge PENDING_KYC (in-flight) | Bridge | Complete existing flow |
| Migrated users | Grid | Re-do grid_kyc step |

### 19.3 Migration script (optional admin tool)

`scripts/migrate-user-bridge-to-grid.ts`:

```
For each user with BridgeCustomer.status=ACTIVE:
  1. Ensure KycProfile exists and Sumsub GREEN
  2. GridKycHandler.createOrUpdateCustomer(userId)
  3. Generate grid kyc-link → user must complete grid_kyc
  4. For each BridgeExternalAccount:
     - GridExternalAccountHandler.createFromBridgeAccount(...)
  5. Set user metadata.payoutProvider = 'grid'
  6. Do NOT delete Bridge records
```

### 19.4 API route deprecation timeline

| Phase | Bridge endpoints | Grid endpoints |
|-------|------------------|----------------|
| Dev | Active | Active |
| Beta | Active (legacy users) | Active (new users) |
| GA | Read-only status/history | Primary |
| GA+90d | Return 410 for new prepare/execute | Primary |

Deprecated Bridge endpoints (eventually):
- `POST /bridge/payout/prepare`
- `POST /bridge/payout/execute`
- `POST /bridge/transfers/payout`
- `POST /bridge/external-accounts` (for payout payees)

Retained Bridge endpoints:
- `GET /bridge/off-ramp/:id/status` (history)
- `GET /bridge/external-accounts` (history)
- On-ramp endpoints (out of scope — separate decision)

### 19.5 Data retention

| Table | Retention |
|-------|-----------|
| `BridgeTransaction` | Forever (audit) |
| `GridTransaction` | Forever |
| `BridgeCustomer` | Forever |
| `GridCustomer` | Forever |

---

## 20. Testing Strategy

### 20.1 Sandbox E2E — happy path (CLABE)

```
1. GRID_ENABLED=true, sandbox credentials
2. Create test user with KycProfile (magic lastName → APPROVED in sandbox)
3. POST /customers → Customer:...
4. POST /kyc-link → skip in sandbox (auto-approved)
5. POST /sandbox/internal-accounts/{id}/fund { amount: 10000, currency: USD }
6. POST /external-accounts { SPEI, clabe: ... }
7. POST /payout/prepare { amount: 500, currency: MXN }
8. Sandbox path C: skip Relay, use ACCOUNT source
9. POST /payout/execute
10. Assert GridTransaction.status → COMPLETED
11. Assert webhook received
```

### 20.2 Sandbox E2E — Relay funding path

```
1. POST /payout/prepare → capture relay steps + depositAddress
2. Simulate transactionHash (mock Relay in test env)
3. POST /payout/execute
4. Mock incoming_payment.received webhook
5. Assert executeQuote called
6. Mock transaction.status_change → COMPLETED
```

### 20.3 Unit tests

| File | Tests |
|------|-------|
| `grid-account-type.mapper.spec.ts` | All Bridge→Grid type mappings |
| `grid-kyc-payload.mapper.spec.ts` | KycProfile→Grid customer payload |
| `grid-webhook-verify.util.spec.ts` | Signature verification, replay rejection |
| `grid-payout.handler.spec.ts` | Prepare validation, quote expiry, status transitions |
| `grid-kyc.handler.spec.ts` | Skip duplicate customer, kyc-link generation |

### 20.4 Integration tests (e2e)

`test/grid-payout.flow.e2e-spec.ts`:
- Auth + KYC gating
- External account CRUD
- Prepare rejects without KYC
- Webhook updates status

### 20.5 Sandbox magic values

| Field | Value | Result |
|-------|-------|--------|
| `lastName` ends `001` | Test User 001 | KYC PENDING |
| `lastName` ends `002` | Test User 002 | KYC REJECTED |
| Other | Test User | KYC APPROVED |

### 20.6 Test webhook

```bash
curl -X POST "$GRID_BASE_URL/sandbox/webhooks/send-test" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"eventType": "transaction.status_change"}'
```

---

## 21. Production Checklist

### 21.1 Pre-launch

- [ ] Production Grid credentials from Lightspark
- [ ] `PATCH /platform` with production `webhookUrl` and currencies
- [ ] `GRID_WEBHOOK_SECRET` in AWS secrets
- [ ] Webhook HMAC verification tested against real Grid events
- [ ] Sandbox E2E all target corridors: USD, MXN, EUR, BRL, GBP
- [ ] JIT crypto funding spike completed (§10.2)
- [ ] Relay + Grid funding E2E on staging
- [ ] Quote expiration UX tested on slow mobile networks
- [ ] Idempotency keys verified (duplicate execute safe)
- [ ] Transaction history shows Grid payouts
- [ ] Push notifications on COMPLETED
- [ ] Error messages user-tested
- [ ] Rate limits understood (1000 req/min)
- [ ] Logging + alerting on `GridTransaction.status=FAILED`
- [ ] Compliance review with Lightspark
- [ ] BYO KYC request submitted (optional v2)

### 21.2 Launch

- [ ] `GRID_ENABLED=true` in production
- [ ] Feature flag: new signups → `payoutProvider=grid`
- [ ] Monitor first 100 payouts manually
- [ ] Dashboard: Grid transaction failure rate < 1%

### 21.3 Rollback

If critical failure:
1. Set `GRID_ENABLED=false`
2. Set `PAYOUT_PROVIDER=bridge` for all users
3. Bridge module still operational for legacy users
4. Grid transactions in-flight complete via webhooks (no cancel API assumed)

---

## 22. Frontend Integration Guide

### 22.1 Feature detection

```typescript
const config = await api.get('/app-config');
const payoutProvider = config.payoutProvider; // 'grid' | 'bridge'
const basePath = payoutProvider === 'grid' ? '/grid' : '/bridge';
```

### 22.2 KYC checklist — Grid partner

New step `grid_kyc` appears in `GET /kyc/status/grid`:

```
1. id_verification (Sumsub — existing)
2. questionnaire (existing)
3. grid_kyc (NEW — redirect to Grid kyc-link URL)
```

**grid_kyc flow:**
```typescript
const { url } = await api.post('/grid/kyc/link');
// Open WebView with url
// On deep link anzo://kyc/grid-complete → refresh status
```

### 22.3 Payout flow (unchanged UX)

```typescript
// 1. Select payee
const payees = await api.get('/grid/external-accounts');

// 2. Get fee preview
const preview = await api.post('/grid/payout/quote', {
  amount: '100.00',
  currency: 'MXN',
  amountType: 'receive',
});

// 3. Prepare
const prepare = await api.post('/grid/payout/prepare', {
  externalAccountId: payee.id,
  amount: '100.00',
  currency: 'MXN',
});

// 4. Show rate/fee confirmation
if (prepare.amountAdjusted) {
  await confirm(`Paying out ${prepare.amount} instead of ${prepare.requestedAmount}`);
}

// 5. Sign Relay steps (SAME as Bridge)
const txHash = await turnkeySignAndBroadcast(prepare.steps);

// 6. Execute
await api.post('/grid/payout/execute', {
  gridPayoutId: prepare.gridPayoutId,
  transactionHash: txHash,
  requestId: prepare.requestId,
});

// 7. Poll status
poll(() => api.get(`/grid/payout/${prepare.gridPayoutId}/status`), {
  until: (s) => ['COMPLETED', 'FAILED', 'QUOTE_EXPIRED'].includes(s.status),
});
```

### 22.4 Status display mapping

| Status | UI |
|--------|-----|
| `DEPOSIT_BROADCASTING` | "Processing your transfer..." |
| `DEPOSIT_CONFIRMED` | "Deposit confirmed, sending to bank..." |
| `EXECUTING` / `IN_FLIGHT` | "Payment sent to bank" |
| `COMPLETED` | "Complete!" |
| `FAILED` | Show `errorMessage` |
| `QUOTE_EXPIRED` | "Rate expired — tap to retry" |

### 22.5 Add payee forms

Reuse existing Bridge payee forms — same DTO shape. Only API path changes to `/grid/external-accounts`.

Supported countries (v1 target): US, MX, BR, GB, EU (SEPA), IN (UPI).

---

## 23. Open Questions & Lightspark Confirmations

| # | Question | Impact | Owner |
|---|----------|--------|-------|
| Q1 | Does JIT quote return **USDC-on-Base deposit address** in `paymentInstructions`? | Blocks Funding Model A without fallback | Eng spike |
| Q2 | Exact field names in `paymentInstructions` for crypto? | Relay adapter parsing | OpenAPI review |
| Q3 | Is **BYO KYC** available with Anzo Sumsub `applicantId` or share token? | Eliminates double KYC | Lightspark account manager |
| Q4 | GB Faster Payments — exact `accountType` enum value? | GB corridor | OpenAPI |
| Q5 | Can `platformCustomerId` be Anzo `userId` (cuid format)? | Customer creation | Sandbox test |
| Q6 | Max payout amounts per corridor? | Validation rules | Grid dashboard |
| Q7 | Developer fee — does Grid support platform fee on top? | `GRID_TRANSFER_DEVELOPER_FEE` | Lightspark |
| Q8 | Webhook `X-Grid-Signature` — `sha256=` prefix always present? | Guard implementation | Sandbox test |
| Q9 | Quote re-create after deposit but before execute — atomic? | Expiry edge case | Lightspark support |
| Q10 | India UPI — production corridor availability date? | INR support | Lightspark |

**Spike ticket (Sprint 0):** Sandbox JIT quote USD→MXN with `sourceType: JIT`; log full `paymentInstructions` JSON.

---

## 24. Implementation Phases

### Phase 0 — Spike (3–5 days)

- [ ] Grid sandbox account + credentials
- [ ] Manual curl: customer → fund → external account → quote → execute (CLABE)
- [ ] JIT quote: capture `paymentInstructions` for crypto
- [ ] Confirm webhook delivery to ngrok
- [ ] Document actual field names in spike addendum

### Phase 1 — Foundation (1 week)

- [ ] Prisma migration: Grid tables
- [ ] `GridProvider` with customer + accounts + quotes
- [ ] `GRID_*` env vars + validation
- [ ] `GridModule` registered in `app.module.ts`
- [ ] `GridWebhookGuard` + controller skeleton

### Phase 2 — KYC (1 week)

- [ ] `grid` partner in `partner-steps.config.ts`
- [ ] `grid-kyc-payload.mapper.ts`
- [ ] `GridKycHandler` + `checkAndSubmitGrid`
- [ ] `GET /grid/kyc/status`, `POST /grid/kyc/link`
- [ ] Webhook: `verification.status_change`, `customer.status_change`
- [ ] Internal account sync on KYC approval

### Phase 3 — External accounts (3–4 days)

- [ ] `GridExternalAccountHandler`
- [ ] DTOs (reuse Bridge shapes)
- [ ] CRUD endpoints
- [ ] `grid-account-type.mapper.ts` + tests

### Phase 4 — Payouts (1.5 weeks)

- [ ] `GridPayoutHandler` — prepare/execute/status/quote
- [ ] Relay adapter (§10)
- [ ] `incoming_payment.received` → auto-execute
- [ ] `transaction.status_change` → completion
- [ ] Transaction history integration
- [ ] Push notifications

### Phase 5 — QA & staging (1 week)

- [ ] E2E tests
- [ ] All corridors in sandbox
- [ ] Load test webhook handler
- [ ] Mobile integration with staging API

### Phase 6 — Production beta (1 week)

- [ ] Production credentials
- [ ] Limited user rollout
- [ ] Monitor + fix

### Phase 7 — Bridge sunset (ongoing)

- [ ] Migration script for willing users
- [ ] Deprecate Bridge payout endpoints
- [ ] Update `docs/SEAMLESS_PAYOUT_FLOW.md` → point to Grid

---

## 25. Reference Links

### Lightspark Grid

| Resource | URL |
|----------|-----|
| Grid product page | https://www.lightspark.com/grid |
| Documentation | https://docs.lightspark.com/ |
| OpenAPI spec | https://docs.lightspark.com/openapi.yaml |
| LLMs.txt | https://docs.lightspark.com/llms.txt |
| GitHub (grid-api) | https://github.com/lightsparkdev/grid-api |
| Dashboard | https://app.lightspark.com |
| Payouts & B2B | https://docs.lightspark.com/payouts-and-b2b |
| Payouts quickstart | https://docs.lightspark.com/payouts-and-b2b/quickstart |

### Local documentation mirror

| Path | Content |
|------|---------|
| `docs/lightspark-grid-docs/00-index.md` | TOC |
| `docs/lightspark-grid-docs/05-customers/` | Customer API |
| `docs/lightspark-grid-docs/04-kyc-kyb-sumsub/` | KYC/KYB + Sumsub |
| `docs/lightspark-grid-docs/07-payouts-and-b2b/` | Payout flows |
| `docs/lightspark-grid-docs/08-ramps/` | Fiat↔crypto ramps |
| `docs/lightspark-grid-docs/13-webhooks/` | Webhook events |
| `docs/lightspark-grid-docs/99-end-to-end-flow/` | Full integration guide |

### Anzo internal

| Resource | Path |
|----------|------|
| Prisma schema | `prisma/schema.prisma` |
| Modular KYC | `docs/MODULAR_KYC_SYSTEM.md` |
| KYC architecture | `docs/KYC_SUMSUB_ARCHITECTURE.md` |
| Bridge payout impl | `docs/WALLET_PAYOUT_IMPLEMENTATION.md` |
| Seamless payout flow | `docs/SEAMLESS_PAYOUT_FLOW.md` |
| Bridge payout handler | `src/modules/bridge/handlers/bridge-payout.handler.ts` |
| Partner submission | `src/modules/kyc/handlers/partner-submission.handler.ts` |
| KYC partner steps | `src/modules/kyc/config/partner-steps.config.ts` |

---

## Appendix A — Full Grid API Quick Reference

### Platform
- `GET /platform`
- `PATCH /platform`

### Customers
- `POST /customers`
- `GET /customers/{id}`
- `PATCH /customers/{id}`
- `DELETE /customers/{id}`
- `GET /customers`
- `POST /customers/bulk-upload`
- `GET /customers/bulk-upload/{jobId}`
- `POST /customers/{customerId}/kyc-link`

### Verifications & documents
- `POST /verifications`
- `GET /verifications/{id}`
- `GET /verifications`
- `POST /beneficial-owners`
- `GET /beneficial-owners/{id}`
- `PATCH /beneficial-owners/{id}`
- `GET /beneficial-owners`
- `POST /documents` (multipart)
- `GET /documents/{id}`
- `GET /documents`
- `POST /documents/{id}/replace`
- `DELETE /documents/{id}`

### Accounts
- `GET /internal-accounts`
- `GET /internal-accounts/{id}`
- `PATCH /internal-accounts/{id}`
- `POST /internal-accounts/{id}/export`
- `POST /customers/{customerId}/external-accounts`
- `GET /customers/{customerId}/external-accounts`
- `GET /customers/{customerId}/external-accounts/{id}`
- `DELETE /customers/{customerId}/external-accounts/{id}`
- `POST /platform/external-accounts`
- `GET /platform/external-accounts`
- `GET /external-accounts/{id}/lookup`

### Transfers
- `POST /quotes`
- `GET /quotes/{id}`
- `POST /quotes/{id}/execute`
- `GET /quotes/estimate-crypto-fee`
- `POST /transfer-ins`
- `POST /transfer-outs`

### Transactions
- `GET /transactions`
- `GET /transactions/{id}`
- `POST /transactions/{id}/approve`
- `POST /transactions/{id}/reject`
- `POST /transactions/{id}/confirm-receipt`

### Sandbox
- `POST /sandbox/internal-accounts/{id}/fund`
- `POST /sandbox/webhooks/send-test`

### Exchange
- `GET /exchange-rates`
- `GET /discoveries?country=&currency=`

---

## Appendix B — KycProfile → Grid Customer (complete mapper)

```typescript
export function mapKycProfileToGridCustomer(
  profile: KycProfile & { user: User },
): GridCreateCustomerPayload {
  const region = alpha3ToAlpha2(profile.country);
  const nationality = alpha3ToAlpha2(profile.nationality || profile.country);
  const fullName = [profile.firstName, profile.middleName, profile.lastName]
    .filter(Boolean)
    .map((s) => s!.trim())
    .join(' ');

  const payload: GridCreateCustomerPayload = {
    customerType: 'INDIVIDUAL',
    platformCustomerId: profile.userId,
    region,
    currencies: ['USD', 'USDC'],
    fullName,
    birthDate: profile.dateOfBirth!,
    nationality,
    email: profile.user.email,
    phone: profile.phone || undefined,
    address: {
      line1: profile.streetLine1!,
      line2: profile.streetLine2 || undefined,
      city: profile.city!,
      state: profile.state || undefined,
      postalCode: profile.postalCode!,
      country: region,
    },
  };

  if (profile.taxId) {
    payload.identification = {
      type: mapTaxIdToGridIdentification(profile.taxIdType, region),
      number: profile.taxId,
    };
  }

  return payload;
}
```

---

## Appendix C — GridPayoutHandler pseudocode (full)

```typescript
class GridPayoutHandler {
  async preparePayout(userId: string, dto: PrepareGridPayoutDto) {
    const customer = await this.requireActiveGridCustomer(userId);
    const ext = await this.requireExternalAccount(userId, dto.externalAccountId);
    const wallet = await this.requireEvmWallet(userId);

    const quote = await this.gridProvider.createQuote({
      source: { sourceType: 'JIT', customerId: customer.gridCustomerId },
      destination: {
        destinationType: 'ACCOUNT',
        accountId: ext.gridExternalAccountId,
      },
      amount: dto.amount,
      amountCurrency: dto.currency.toUpperCase(),
      amountLock: dto.amountLock ?? 'RECEIVING',
    }, `grid-quote-${randomUUID()}`);

    const funding = parseFundingInstructions(quote);
    let relayQuote = null;
    if (funding) {
      relayQuote = await this.relayProvider.getQuote({
        user: wallet.ethereumAddress,
        originChainId: 8453,
        destinationChainId: funding.chainId,
        originCurrency: USDC_BASE,
        destinationCurrency: USDC_BASE,
        amount: funding.amountAtomic,
        recipient: funding.depositAddress,
        tradeType: 'EXACT_INPUT',
      });
    }

    const tx = await this.prisma.gridTransaction.create({
      data: {
        userId,
        gridCustomerId: customer.id,
        gridQuoteId: quote.id,
        status: 'QUOTED',
        amount: dto.amount,
        destinationCurrency: dto.currency.toLowerCase(),
        sourceAmount: quote.sourceAmount,
        destinationAmount: quote.destinationAmount,
        sourceCurrency: quote.sourceAmountCurrency?.toLowerCase(),
        exchangeRate: quote.exchangeRate,
        fees: quote.fees,
        feesCurrency: quote.feesCurrency,
        depositAddress: funding?.depositAddress,
        metadata: {
          quoteSnapshot: quote,
          payoutContext: { externalAccountId: ext.id, expiresAt: quote.expiresAt },
          fundingInstructions: quote.paymentInstructions,
          relayQuote,
        },
      },
    });

    return {
      gridPayoutId: tx.id,
      quoteId: quote.id,
      steps: relayQuote?.steps ?? [],
      gasless: true,
      depositAddress: funding?.depositAddress,
      amount: dto.amount,
      requestedAmount: dto.amount,
      amountAdjusted: false,
      currency: dto.currency,
      externalAccountId: dto.externalAccountId,
      exchangeRate: quote.exchangeRate,
      fees: quote.fees,
      expiresAt: quote.expiresAt,
    };
  }

  async executePayout(userId: string, dto: ExecuteGridPayoutDto) {
    const tx = await this.requireQuotedTransaction(userId, dto.gridPayoutId);
    const txHash = normalizeEvmTxHash(dto.transactionHash);
    if (!txHash) throw new BadRequestException('transactionHash required');

    await this.prisma.gridTransaction.update({
      where: { id: tx.id },
      data: { status: 'DEPOSIT_BROADCASTING', depositTxHash: txHash },
    });

    this.pollAndExecuteQuote(tx.id, txHash, dto.requestId).catch(/* mark FAILED */);

    return {
      gridPayoutId: tx.id,
      status: 'DEPOSIT_BROADCASTING',
      gasless: true,
      transactionHash: txHash,
      message: 'Payout submitted. Funds will be transferred to your bank account automatically.',
    };
  }

  async pollAndExecuteQuote(txId: string, txHash: string, requestId?: string) {
    const tx = await this.prisma.gridTransaction.findUnique({ where: { id: txId } });
    const meta = tx.metadata as any;

    if (requestId) {
      const result = await this.relayStatus.pollUntilResolved(requestId);
      if (result.status === 'FAILED') throw new Error(result.failReason);
    } else {
      await this.waitForBaseTransactionSuccess(txHash);
    }

    await this.prisma.gridTransaction.update({
      where: { id: txId },
      data: { status: 'DEPOSIT_CONFIRMED' },
    });

    // Wait for Grid funding (webhook may have already fired)
    await this.waitForFunding(txId, 300_000);

    const quote = await this.gridProvider.getQuote(tx.gridQuoteId!);
    if (quote.status === 'EXPIRED') {
      await this.fail(txId, 'QUOTE_EXPIRED');
      return;
    }

    const executed = await this.gridProvider.executeQuote(
      tx.gridQuoteId!,
      `grid-exec-${txId}`,
    );

    await this.prisma.gridTransaction.update({
      where: { id: txId },
      data: {
        status: 'EXECUTING',
        gridTransactionId: executed.id,
      },
    });
  }
}
```

---

## Appendix D — Spec self-review

| Check | Status |
|-------|--------|
| Placeholder scan (TBD/TODO) | Q1–Q10 in §23 are explicit open questions, not silent TBDs |
| Internal consistency | Approach 1 + Funding A consistent throughout |
| Scope | Single implementation plan — Grid payouts only; Bridge on-ramp out of scope |
| Ambiguity | GB accountType flagged Q4; JIT field names flagged Q2 |
| KYC double-verification | Documented; BYO path noted as v2 |
| Mobile compatibility | prepare/execute/status shapes preserved |
| DB migration | Additive only; no Bridge table changes |

---

**Document status:** Ready for implementation planning via `writing-plans` skill.

**Next step for user:** Review this spec. After approval, invoke implementation plan generation.

*End of specification.*
