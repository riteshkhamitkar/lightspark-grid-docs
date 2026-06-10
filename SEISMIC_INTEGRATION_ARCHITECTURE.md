# Seismic Orchestration — Anzo Backend Integration Architecture

> **Purpose:** Implementation architecture for [Seismic Orchestration](https://orchestration.seismic.systems/docs/) in the Anzo backend.  
> **Scope:** Virtual accounts, payouts, KYC/onboarding, quotes/orders, webhooks, database schema, env vars, and migration from Bridge.xyz + Sumsub.  
> **Status:** **APPROVED FOR IMPLEMENTATION** (verified against live official docs, June 10, 2026)  
> **Sources:** Live docs at `https://orchestration.seismic.systems/docs/`, `docs/VIRTUAL_ACCOUNTS_AND_PAYOUTS.md`, `docs/seismic-documentation/*`, full Anzo codebase (`prisma/schema.prisma`, Bridge/KYC/Relay modules), `.env.example`, `validation.schema.ts`

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Live Official Documentation Verification](#2-live-official-documentation-verification)
3. [Documentation Accuracy Audit (Local Mirror)](#3-documentation-accuracy-audit-local-mirror)
4. [Current Anzo Architecture (Baseline)](#4-current-anzo-architecture-baseline)
5. [Seismic Product Model](#5-seismic-product-model)
6. [Gap Analysis: Bridge + Sumsub vs Seismic](#6-gap-analysis-bridge--sumsub-vs-seismic)
7. [Recommended Target Architecture](#7-recommended-target-architecture)
   - [7.4 Critical API Behaviors (Mandatory)](#74-critical-api-behaviors-mandatory--live-docs)
8. [Per-User Account Checklist](#8-per-user-account-checklist-virtual-accounts-study-list)
9. [Payout Countries & Rails Matrix](#9-payout-countries--rails-matrix)
10. [KYC Onboarding Checklist](#10-kyc-onboarding-checklist-study-list)
11. [Database Schema Design](#11-database-schema-design)
12. [Backend Module Design](#12-backend-module-design)
13. [API Surface](#13-api-surface)
14. [Webhooks & Event Processing](#14-webhooks--event-processing)
15. [Environment Variables & Configuration](#15-environment-variables--configuration)
16. [Migration Strategy & Feature Flags](#16-migration-strategy--feature-flags)
17. [End-to-End Example: New User Workflow](#17-end-to-end-example-new-user-workflow)
18. [Risks, Blockers & Open Questions](#18-risks-blockers--open-questions)
19. [Implementation Phases](#19-implementation-phases)
20. [Appendix: Local Docs Corrections](#20-appendix-local-docs-corrections)

---

## 1. Executive Summary

Anzo today runs **fiat on-ramp (virtual accounts)**, **fiat off-ramp (payouts)**, and **modular KYC** through a **dual-provider stack**:

| Layer | Provider | Role |
|-------|----------|------|
| Identity | **Sumsub** | Mobile SDK: ID scan + liveness; OCR prefill |
| Compliance + rails | **Bridge.xyz** | Customer record, TOS, USD/EUR virtual accounts, Bridge wallet, external bank accounts, transfers |
| Crypto movement (payout step 1) | **Relay Protocol** | USDC: user Turnkey wallet → Bridge wallet on Base |
| Canada CAD | **Paytrie** | Parallel partner (Sumsub share-token); **not** Seismic scope |

**Seismic Orchestration** is a unified REST API (+ TypeScript SDK) that can replace Bridge + Sumsub for new flows: KYC/KYB, account provisioning, quote fan-out, order settlement, and Svix webhooks.

### Recommended approach for Anzo

**Hybrid native API** (not a pure Bridge compat base-URL swap):

| Use Seismic native API for | Keep Bridge compat only as optional short-term bridge |
|----------------------------|-----------------------------------------------------|
| Fiat virtual accounts (`POST /customers/:id/accounts/fiat/virtual`) | External account CRUD (temporary) |
| Crypto external account (user Turnkey address on Base) | Transfer shape mapping (temporary) |
| Quotes + order acceptance (`POST /quotes`, accept) | |
| Unified Svix webhooks | |
| Customer + KYC (hosted link **or** Sumsub-retained inline profile) | |

**Why not compat-only:** Bridge compat returns **501** for virtual accounts and Bridge wallets, and **`bridge_wallet` rail is unsupported** — both are core to Anzo's current on-ramp and seamless payout flows.

### Production readiness note

Seismic **sandbox is live**; **production API is listed as "coming soon"** in official docs. Plan implementation and UAT against sandbox; confirm production URL and KYB approval before cutover.

**Implementation start:** Phase 0 (Ops setup) → Phase 1 (Foundation). Phase 1 **must** include `named: "require"` on on-ramp quotes, async `AccountCreationRequest` handling, and automatic `Idempotency-Key` injection in `SeismicProvider`.

---

## 2. Live Official Documentation Verification

Cross-verified against live docs at `https://orchestration.seismic.systems/docs/` (June 10, 2026). This section records confirmed pillars and mandatory refinements.

### 2.1 Verified architectural pillars

| Pillar | Live docs confirmation | Verdict |
|--------|------------------------|---------|
| **Bridge compat is not viable for on-ramp** | Compat supported-endpoint list excludes virtual accounts (`/customers/{id}/virtual_accounts`) and Bridge wallets → **501 `not_implemented`** | ✅ Native API only |
| **Account provisioning endpoints** | `POST /customers/:id/accounts/fiat/virtual`, `POST /customers/:id/accounts/crypto/external` | ✅ `SeismicAccountHandler` mapping correct |
| **Quotes & orders** | Direction-agnostic `POST /quotes` → `AsyncQuoteSnapshot` with `best_quote_id`; accept via `POST /quotes/:id/accept` or SDK `acceptBest()` | ✅ `SeismicQuoteHandler` / `SeismicPayoutHandler` correct |
| **Webhooks (Svix)** | Exactly 5 event types; Svix requires **2xx within 15 seconds** or retries | ✅ Return 200 immediately; process async |
| **Anzo-specific corrections** | EUR VA parity, Turnkey crypto external, "Anzo" naming | ✅ Aligned with non-custodial model |

### 2.2 Mandatory refinements (from live API specs)

These are **not optional** — bake into Phase 1 handlers:

| # | Refinement | Handler(s) | See |
|---|------------|------------|-----|
| R1 | `named: "require"` on fiat→crypto quote requests | `SeismicQuoteHandler` | [§7.4.1](#741-on-ramp-quotes-named-require) |
| R2 | Async `AccountCreationRequest` lifecycle before `ACTIVE` | `SeismicAccountHandler`, `SeismicKycHandler` | [§7.4.2](#742-async-account-creation-lifecycle) |
| R3 | `Idempotency-Key` on all state-mutating calls | `SeismicProvider` | [§7.4.3](#743-idempotency-keys-mandatory) |

---

## 3. Documentation Accuracy Audit (Local Mirror)

Analysis cross-checked `docs/seismic-documentation/*` against live official docs and Anzo codebase behavior.

### Verified accurate in local docs

| Topic | Verdict | Notes |
|-------|---------|-------|
| Sandbox base URL | ✅ | `https://orchestration-sandbox.seismictest.net/api` |
| API path prefix | ✅ | All endpoints under `/api/v0/*` |
| Auth header | ✅ | `Api-Key: seismic_sandbox_...` (mutually exclusive with Bearer) |
| Bridge compat URL | ✅ | `.../api/compat/bridge/v0` |
| Virtual accounts **not** in compat layer | ✅ | Must use native `POST .../accounts/fiat/virtual` |
| Bridge wallets **not** in compat layer | ✅ | Must use native crypto accounts |
| `bridge_wallet` rail unsupported in compat | ✅ | Breaks current seamless payout as-is |
| Quote fan-out + `best_quote_id` | ✅ | Native quotes API |
| Webhooks via Svix (5 event types) | ✅ | `customer.kyc.*`, `account.*`, `order.status_changed` |
| Currency delta vs Bridge | ✅ | Seismic: USD, EUR, GBP, BRL, COP, MXN + USDC only |
| Org KYB required before end-user onboarding | ✅ | Dashboard: `PUT /org/kyb/profile` → `POST /org/kyb/submit` |

### Corrections / gaps in local docs

| Issue | Location | Correction for Anzo |
|-------|----------|---------------------|
| Project name **"Enso"** used throughout | All `seismic-documentation/*`, `seismic-integration-workflow.md` | Should read **Anzo** |
| Default accounts on KYC approval create **USD fiat + USDC crypto virtual only** | `seismic-integration-workflow.md` §5.2 | Anzo must also create **EUR fiat virtual** (parity with Bridge: one USD + one EUR VA per user) |
| Implies Seismic **crypto virtual** on Base as default | Same workflow doc | Anzo should register **Turnkey `ethereumAddress` as crypto external** (non-custodial model); optional Seismic custodial crypto virtual only if product requires a holding wallet |
| Assumes **full Sumsub removal** | Workflow doc Phase 4 | Anzo has deep Sumsub mobile SDK investment — see [§10 KYC paths](#10-kyc-onboarding-checklist-study-list) for retained-Sumsub option |
| `reference_id` on customer create | Mentioned in errors doc | Confirm with Seismic whether `reference_id` = Anzo `userId` for idempotent onboarding |
| Live catalog verification | `GET /rails`, `/currencies` | Verified via live official docs (June 2026); optional sandbox smoke test with API key in CI |
| `llms.txt` / `llms-full.txt` | Official docs redirect | Returned 404 when fetched programmatically; use browser docs + local mirrored references |

### `docs/VIRTUAL_ACCOUNTS_AND_PAYOUTS.md` accuracy

This document is **accurate against the current Anzo codebase** (June 2026). It correctly describes:

- Bridge-only VA provider (USD ACH/Wire + EUR SEPA)
- Sumsub-first modular KYC with 4 Bridge partner steps
- Seamless payout via Relay + Bridge wallet
- Payout currencies: USD, EUR, MXN, BRL, GBP (no COP)
- No Seismic integration exists in repo yet (confirmed: zero `SEISMIC_*` env vars in `validation.schema.ts`)

---

## 4. Current Anzo Architecture (Baseline)

### 3.1 Module map

```
Mobile App
    │
    ▼
Anzo Backend (NestJS, prefix /api/v1)
├── KycModule          Sumsub session, questionnaire, TOS, partner submission
├── BridgeModule       VA, external accounts, payouts, Bridge webhooks
├── RelayModule        USDC transfer quotes (payout prepare)
└── PaytrieModule      Canada CAD (parallel, out of Seismic scope)
```

### 3.2 Key database models (existing)

| Model | Purpose |
|-------|---------|
| `KycProfile` | Anzo-owned PII + Sumsub fields + Bridge questionnaire |
| `KycVerification` | Per-partner step state (`bridge`, `paytrie`) |
| `BridgeCustomer` | 1:1 user ↔ Bridge customer; lifecycle `PENDING_KYC` → `ACTIVE` |
| `BridgeWallet` | Bridge custodial wallet on Base (payout hub) |
| `BridgeVirtualAccount` | USD + EUR deposit accounts; unique `(bridgeCustomerId, sourceCurrency)` |
| `BridgeExternalAccount` | User payout bank accounts |
| `BridgeTransaction` | Off-ramp / seamless payout lifecycle |
| `BridgeActivityEvent` | Activity feed (deposits, KYC, payouts) |

### 3.3 Current virtual account creation (code truth)

Virtual accounts are **never user-initiated**. They are created in `BridgeKycHandler.handleKycApproval()`:

1. Create `BridgeWallet` (`chain: base`)
2. Create USD VA → destination = user Turnkey `ethereumAddress` (or Solana for legacy)
3. Create EUR VA → same destination
4. Set `BridgeCustomer.status = ACTIVE` only when **both** VAs exist

Source: `src/modules/bridge/handlers/bridge-kyc.handler.ts` lines 542–657.

### 3.4 Current KYC gating (frontend contract)

`KycService.computeBridgeOverallStatus()` keeps overall status **`in_progress`** until:

- `BridgeCustomer.status === 'ACTIVE'`, **and**
- At least one `BridgeVirtualAccount` exists

Source: `src/modules/kyc/kyc.service.ts` lines 349–376.

### 3.5 Current environment variables (fiat/KYC)

From `validation.schema.ts` and `.env.example`:

| Variable | Required | Purpose |
|----------|----------|---------|
| `BRIDGE_API_KEY` | Optional (gradual rollout) | Bridge API |
| `BRIDGE_API_URL` | Default `https://api.bridge.xyz/v0` | Bridge base URL |
| `BRIDGE_WEBHOOK_PUBLIC_KEY` | Optional | RSA webhook verify |
| `BRIDGE_DEVELOPER_FEE_PERCENT` | Default `0.0001` | VA developer fee |
| `BRIDGE_TRANSFER_DEVELOPER_FEE` | Default `0.00` | Flat USD transfer fee |
| `SUMSUB_*` | Optional | Sumsub KYC |
| `PAYTRIE_*` | Optional | Canada |
| `BASE_RPC_URL` / `RPC_URL_BASE` | Optional | Payout deposit confirmation |

**No Seismic variables exist today.**

---

## 5. Seismic Product Model

### 4.1 Identity tiers (do not confuse)

```
Tier 2: Org (Anzo the company)     → /org/*, KYB in dashboard
Tier 3: Org admin (dashboard user)  → /auth/* (not needed for backend)
Tier 4: End-customer (Anzo app user) → /customers/*, /quotes/*
```

Backend uses **org-scoped API key** only. All Tier-4 operations are on behalf of app users.

### 4.2 Core primitives

| Primitive | Description |
|-----------|-------------|
| **Customer** | Individual or business; polymorphic `type` + `profile` |
| **Account** | Fiat or crypto; virtual (Seismic-managed) or external (registered) |
| **Quote snapshot** | Fan-out pricing across rails; includes `best_quote_id` |
| **Order** | Created when quote accepted; lifecycle through webhooks |
| **Hosted KYC link** | Auto-minted on `POST /customers`; URL for redirect flow |

### 4.3 API surfaces

| Surface | Base path | Use |
|---------|-----------|-----|
| **Native** | `/api/v0/*` | Recommended for all new Anzo code |
| **Bridge compat** | `/api/compat/bridge/v0/*` | Transfers, external accounts, customers (Bridge-shaped); **not** VAs/wallets |

### 4.4 SDK

- Package: `seismic-orchestration` (npm)
- Runtime: Node ≥ 18, single dep `zod`
- Factory: `SeismicOrchestrationClient.sandbox({ apiKey })` / `.production()`

---

## 6. Gap Analysis: Bridge + Sumsub vs Seismic

### 5.1 Feature mapping

| Anzo feature today | Bridge + Sumsub | Seismic native | Seismic Bridge compat | Migration note |
|--------------------|-----------------|----------------|----------------------|----------------|
| Mobile ID + liveness | Sumsub SDK | Hosted KYC link **or** keep Sumsub + inline profile | KYC links (`POST /kyc_links`) | Product decision: mobile UX vs unified provider |
| Questionnaire | Anzo form → Bridge payload | `PUT /customers/:id/kyc/profile` | Bridge customer PUT | Can keep Anzo form, map to Seismic profile |
| TOS | Bridge WebView | Hosted KYC or `attestations.terms_accepted` | TOS links | Simplify to hosted flow or attestations |
| USD/EUR virtual accounts | Bridge VA API | `POST .../accounts/fiat/virtual` | **501 not implemented** | **Must use native** |
| Custodial holding wallet | `BridgeWallet` | Crypto virtual **or** external | **501 not implemented** | Register Turnkey external; or Seismic crypto virtual if holding needed |
| On-ramp destination | User Turnkey on Base | Crypto **external** = Turnkey address | N/A | Preserves non-custodial model |
| External payout banks | Bridge external accounts | `POST .../accounts/fiat/external` | Implemented | Map existing schemas |
| Seamless payout | Relay → Bridge wallet → transfer | Quote off-ramp + `funding_instructions` / order flow | Transfers (no `bridge_wallet`) | **Redesign payout** around quotes/orders |
| Deposit activity feed | `virtual_account.activity` webhook | `order.status_changed` + `account.updated` | Partial | Remap activity events |
| Multi-rail quote comparison | Manual | Automatic fan-out | Collapsed in `POST /transfers` | Prefer native quotes for UX |
| Colombia COP payouts | Not supported | Supported (Seismic) | Supported | **New market** if enabled |
| Canada CAD | Paytrie | Not in Seismic scope | N/A | Keep Paytrie unchanged |

### 5.2 Critical blockers for compat-only migration

1. **Virtual accounts:** compat returns 501 → on-ramp broken without native API.
2. **`bridge_wallet` rail:** compat returns 422 → seamless payout pattern must change.
3. **Webhook shape:** Bridge RSA vs Seismic Svix → new handler required either way.
4. **Production:** not yet available on Seismic side.

### 5.3 Currency support delta

| | Anzo (Bridge) today | Seismic |
|--|---------------------|---------|
| Fiat VA (deposit) | USD, EUR | USD, EUR (via native fiat virtual) |
| Payout fiat | USD, EUR, MXN, BRL, GBP | USD, EUR, GBP, BRL, MXN, **COP** |
| Stablecoin | USDC (Base/Solana) | **USDC only** (no USDT, DAI, etc.) |

---

## 7. Recommended Target Architecture

### 6.1 High-level diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           Anzo Mobile App                                 │
│  KYC (hosted link OR Sumsub SDK) → VA screen → Payout → Activity feed    │
└──────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        Anzo Backend (NestJS)                              │
│                                                                           │
│  ┌─────────────┐   ┌──────────────────┐   ┌─────────────────────────┐  │
│  │  KycModule  │   │  SeismicModule   │   │  BridgeModule (legacy)  │  │
│  │  (optional  │──▶│  SeismicProvider │   │  feature-flagged        │  │
│  │   Sumsub)   │   │  SeismicService  │   │  during migration       │  │
│  └─────────────┘   │  WebhookHandler  │   └─────────────────────────┘  │
│                    │  AccountProvisioner│                                │
│  ┌─────────────┐   │  QuoteHandler    │   ┌─────────────────────────┐  │
│  │ RelayModule │◀──│  PayoutHandler   │   │  PaytrieModule (unchanged)│  │
│  └─────────────┘   └────────┬─────────┘   └─────────────────────────┘  │
└─────────────────────────────┼────────────────────────────────────────────┘
                              │ Api-Key
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                  Seismic Orchestration (Native API)                       │
│  Customers · KYC · Accounts · Quotes · Orders · Svix Webhooks            │
└──────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Design principles

1. **Provider abstraction at the service boundary** — `FiatRailsProvider` interface with `BridgeFiatRailsProvider` and `SeismicFiatRailsProvider` implementations; controllers stay stable where possible.
2. **Preserve non-custodial wallets** — on-ramp USDC lands on Turnkey `ethereumAddress` via crypto external account registration, not Seismic custodial wallet (unless product explicitly wants custodial).
3. **Parity with current VA set** — every approved user gets **USD + EUR fiat virtual accounts** (same as Bridge today).
4. **Quotes-first for money movement** — replace direct Bridge transfers with `POST /quotes` → accept → order tracking.
5. **Feature flags** — dual-run Bridge + Seismic during migration; route by user cohort.
6. **Paytrie untouched** — Canada remains separate.

### 7.3 Payout architecture change (important)

**Today (seamless payout):**

```
User Turnkey USDC ──Relay──▶ Bridge Wallet ──Bridge transfer──▶ External bank
```

**Target (Seismic native):**

```
Option A (recommended):
  POST /quotes { source: USDC/base (Turnkey external acct), dest: EUR/sepa (external bank) }
  → accept → order
  → funding_instructions tell user where to send USDC (or Relay-assisted broadcast)
  → order.status_changed webhooks drive UI

Option B (if Seismic crypto virtual required as hub):
  Relay → Seismic crypto virtual → quote off-ramp → fiat external
  (Only if quote engine requires custodial source account)
```

**Action:** Confirm with Seismic sandbox whether off-ramp quotes accept **crypto external** (Turnkey) as source without a custodial intermediate account. This determines whether Relay stays in the payout path.

### 7.4 Critical API Behaviors (Mandatory — Live Docs)

These three behaviors are **required in Phase 1** implementations. They are not edge cases.

#### 7.4.1 On-ramp quotes: `named: "require"`

When requesting an **on-ramp quote** (Fiat → Crypto), the live `QuoteRequest` supports a `named` parameter. Set:

```json
{
  "end_customer_id": "<seismic-customer-uuid>",
  "end_customer_type": "individual",
  "amount": "1000",
  "named": "require",
  "source": {
    "currency": "USD",
    "rails": ["ach", "wire"]
  },
  "destination": {
    "currency": "USDC",
    "rails": ["base"],
    "account": "<turnkey-crypto-external-account-id>"
  }
}
```

**Why it matters:** `named: "require"` restricts quote options to funding instructions backed by **named virtual accounts** — the beneficiary name on the VA matches the verified customer's legal name. Without this, Seismic may return generic funding instructions; the user's sending bank can reject ACH/Wire because the beneficiary name does not match.

**Rule:** `SeismicQuoteHandler.createOnrampQuote()` **must** pass `named: "require"` for all fiat→crypto flows. Do **not** set it on off-ramp or swap quotes.

#### 7.4.2 Async account creation lifecycle

The live API distinguishes:

| Resource | When it exists |
|----------|----------------|
| **`AccountCreationRequest`** | Returned immediately from `POST .../accounts/fiat/virtual` with `status: "pending"` |
| **`Account`** | Created asynchronously when provisioning completes |

**Implication:** Calling `POST .../fiat/virtual` does **not** instantly yield deposit instructions. The usable `Account` row appears later.

**Provisioning flow (recommended — webhook-driven):**

```
customer.kyc.status_changed (approved)
  → lifecycleStatus = PROVISIONING_ACCOUNTS
  → POST fiat/virtual (USD) + POST fiat/virtual (EUR) + POST crypto/external (Turnkey)
  → Persist SeismicAccountCreationRequest rows (status: pending) for each
  → WAIT — do NOT set lifecycleStatus = ACTIVE yet

account.created webhook (USD VA)
  → Upsert SeismicAccount (status: active, details: deposit instructions)
account.created webhook (EUR VA)
  → Upsert SeismicAccount
account.created webhook (crypto external) OR immediate if sync
  → Upsert SeismicAccount (purpose: wallet_destination)

When ALL required accounts present AND status = active:
  → lifecycleStatus = ACTIVE
  → Push + SEISMIC_BANK_ACCOUNTS_READY activity
```

**Fallback (polling):** If `account.created` is delayed, poll `GET /customers/:id/accounts/requests/:request_id` until `status === "completed"`, then upsert `SeismicAccount` from `account_id`.

**Gap-fill:** Mirror Bridge's gap-fill pattern — status polling and webhook both call `tryActivateCustomer(userId)` which checks readiness before promoting to `ACTIVE`.

#### 7.4.3 Idempotency keys (mandatory)

Live docs require `Idempotency-Key` on **all state-mutating** endpoints. Keys expire after 24 hours; same key + same payload → same response; different payload + same key → 409.

**`SeismicProvider` must auto-inject keys** — never leave this to individual handlers:

| Operation | Idempotency-Key pattern |
|-----------|-------------------------|
| `POST /customers` | `customer-{anzoUserId}` |
| `POST .../accounts/fiat/virtual` (USD) | `va-usd-{seismicCustomerId}` |
| `POST .../accounts/fiat/virtual` (EUR) | `va-eur-{seismicCustomerId}` |
| `POST .../accounts/crypto/external` | `wallet-base-{seismicCustomerId}` |
| `POST .../accounts/fiat/external` | `ext-{seismicCustomerId}-{hash}` |
| `POST /quotes` | `quote-{clientRequestId}` |
| `POST /quotes/:id/accept` | `accept-{snapshotId}` |
| `POST /quotes/:id/accept-best` | `accept-best-{snapshotId}` |

**Why critical for webhooks:** If Svix retries `customer.kyc.status_changed` and handler re-runs provisioning, idempotency keys prevent duplicate USD/EUR virtual accounts for the same customer.

```typescript
// SeismicProvider — every mutating call goes through this wrapper
async mutate<T>(
  method: 'POST' | 'PUT' | 'PATCH' | 'DELETE',
  path: string,
  body: unknown,
  idempotencyKey: string,
): Promise<T> {
  return this.http.request({
    method,
    path,
    body,
    headers: { 'Idempotency-Key': idempotencyKey },
  });
}
```

---

## 8. Per-User Account Checklist (Virtual Accounts Study List)

For **each Anzo user who completes KYC and activates fiat features**, the following accounts should exist in Seismic and be mirrored in Anzo DB.

### 8.1 Automatically provisioned (post KYC `approved`)

> **Async:** Each `POST .../accounts/fiat/virtual` returns an `AccountCreationRequest` (`status: "pending"`). Deposit instructions are available only after `account.created` (or poll until `completed`). See [§7.4.2](#742-async-account-creation-lifecycle).

| # | Seismic account type | API | Currency | Shape / Rails | Purpose | Anzo DB model |
|---|---------------------|-----|----------|---------------|---------|---------------|
| 1 | **Fiat virtual** | `POST /customers/:id/accounts/fiat/virtual` | **USD** | `us_bank` → `ach`, `wire` | Receive ACH/Wire deposits; on-ramp | `SeismicAccount` `kind=fiat`, `purpose=onramp_deposit` |
| 2 | **Fiat virtual** | Same | **EUR** | `iban` → `sepa` | Receive SEPA deposits; on-ramp | Same |
| 3 | **Crypto external** | `POST /customers/:id/accounts/crypto/external` | **USDC** | `evm` → `base` | Receive converted USDC to Turnkey address | `SeismicAccount` `kind=crypto`, `purpose=wallet_destination` |

**Not auto-created (user-initiated):**

| # | Seismic account type | API | When | Purpose |
|---|---------------------|-----|------|---------|
| 4 | **Fiat external** | `POST .../accounts/fiat/external` | User adds payout bank | Off-ramp destination |
| 5 | **Crypto external** (optional) | Same | If user adds alternate chain | Future multi-chain |

### 8.2 Optional (only if product requires custodial hub)

| # | Account | When | Replaces |
|---|---------|------|----------|
| 6 | **Crypto virtual** (Seismic-managed USDC on Base) | If quotes require custodial source | `BridgeWallet` |

**Recommendation:** Start without #6; register Turnkey as crypto external (#3). Add custodial crypto virtual only if sandbox quote tests fail without it.

### 8.3 Parity checklist vs Bridge (must match)

| Bridge today | Seismic target | Status |
|--------------|----------------|--------|
| 1× USD VA | 1× USD fiat virtual | Required |
| 1× EUR VA | 1× EUR fiat virtual | Required |
| 1× BridgeWallet (Base) | Turnkey crypto external **or** crypto virtual | Design choice |
| N× External accounts | N× fiat external | Required |
| VA → USDC → Turnkey address | Order/on-ramp → USDC → Turnkey external | Required |

### 8.4 User "ready" gating (mirror current behavior)

Equivalent of `computeBridgeOverallStatus`. **Do not set `ACTIVE` on KYC approval alone** — wait for async account provisioning to complete.

**Lifecycle states:**

```
PENDING_KYC → (kyc approved) → PROVISIONING_ACCOUNTS → (all accounts active) → ACTIVE
                                    ↓ failure/retry
                                 KYC_APPROVED (stuck — gap-fill retries provisioning)
```

**Activation predicate (`tryActivateCustomer`):**

```
seismicReady =
  SeismicCustomer.kycStatus === 'approved'
  AND SeismicCustomer.lifecycleStatus !== 'ACTIVE'  // idempotent guard
  AND EXISTS SeismicAccount WHERE purpose='onramp_deposit' AND currency='USD' AND status='active'
  AND EXISTS SeismicAccount WHERE purpose='onramp_deposit' AND currency='EUR' AND status='active'
  AND EXISTS SeismicAccount WHERE purpose='wallet_destination' AND status='active'
```

When `seismicReady` → set `lifecycleStatus = 'ACTIVE'` → emit `SEISMIC_BANK_ACCOUNTS_READY`.

**Frontend overall status:** Keep `in_progress` while `lifecycleStatus === 'PROVISIONING_ACCOUNTS'` (same UX as Bridge waiting for VAs).

---

## 9. Payout Countries & Rails Matrix

### 8.1 Supported payout destinations

| Country | Currency | Account shape | Payment rail | Anzo today | Seismic | Amount mode |
|---------|----------|---------------|--------------|------------|---------|-------------|
| United States | USD | `us` | `ach`, `wire` | ✅ | ✅ | Send amount |
| Eurozone | EUR | `iban` | `sepa` | ✅ | ✅ | Fixed receive |
| United Kingdom | GBP | `uk` | `faster_payments` | ✅ | ✅ | Fixed receive |
| Mexico | MXN | `mx_clabe` | `spei` | ✅ | ✅ | Fixed receive |
| Brazil | BRL | `br_pix` | `pix` | ✅ | ✅ | Fixed receive |
| Colombia | COP | `co_bank` | `co_bank_transfer` | ❌ | ✅ | TBD — **new** |
| India | INR | `in_upi` | `upi` | ❌ | Documented shape | **Not in Anzo schemas** |
| Canada | CAD | — | — | Paytrie only | — | Out of scope |

### 8.2 Virtual account deposit rails (on-ramp)

| Currency | Deposit rails | Crypto out | Destination |
|----------|---------------|------------|-------------|
| USD | ACH push, Wire | USDC | Base (Turnkey address) |
| EUR | SEPA | USDC | Base (Turnkey address) |

### 8.3 KYC-eligible countries

- **Bridge today:** Dynamic list via `GET /lists/countries` → exposed as `GET /api/v1/kyc/config/countries`
- **Seismic:** Driven by org KYB + customer KYC provider; no direct Bridge `listCountries` equivalent in native API
- **Action:** Cache Seismic catalog + confirm with Seismic which residential countries are accepted for individual KYC under Anzo's org KYB

---

## 10. KYC Onboarding Checklist (Study List)

Two supported paths — **pick one for MVP** (team decision):

### Path A — Seismic hosted KYC (simplest provider count)

| Step | Actor | Action | Seismic API / event |
|------|-------|--------|---------------------|
| 0 | Anzo ops | Complete org KYB in Seismic dashboard | `PUT /org/kyb/profile`, `POST /org/kyb/submit` |
| 1 | User | Tap "Get bank account" | — |
| 2 | Backend | Create Seismic customer | `POST /customers` |
| 3 | Backend | Persist `SeismicCustomer` row | DB |
| 4 | User | Open hosted KYC URL (`kyc.link.url`) | Seismic UI: ID, selfie, questionnaire, TOS |
| 5 | Seismic | Review | — |
| 6 | Backend | Receive webhook | `customer.kyc.status_changed` |
| 7 | Backend | On `approved`: start provisioning (`PROVISIONING_ACCOUNTS`) | See [§7.4.2](#742-async-account-creation-lifecycle) |
| 8 | Backend | `account.created` × 3 → `tryActivateCustomer()` → `ACTIVE` | Push + `SEISMIC_BANK_ACCOUNTS_READY` |
| 9 | User | View deposit instructions | `GET /api/v1/seismic/virtual-accounts` |

**Checklist UI steps (frontend):**

1. Identity verification (hosted link) — **required**
2. Processing — **automatic**
3. Bank accounts ready — **automatic**

### Path B — Retain Sumsub + inline Seismic profile (preserves mobile SDK)

| Step | Actor | Action |
|------|-------|--------|
| 1 | User | Sumsub SDK (existing) | `POST /api/v1/kyc/sumsub/session` |
| 2 | Backend | OCR prefill → `KycProfile` | Sumsub webhook |
| 3 | User | Anzo questionnaire (existing) | `POST /api/v1/kyc/questionnaire` |
| 4 | Backend | Map `KycProfile` → Seismic | `PUT /customers/:id/kyc/profile` |
| 5 | Backend | Submit | `POST /customers/:id/kyc/submit` |
| 6 | Backend | Poll or webhook | `customer.kyc.status_changed` |
| 7+ | Same as Path A steps 7–9 | | |

**Trade-off:** Two verification vendors during transition unless Sumsub is removed after Seismic approves.

### Partner steps config today (`partner-steps.config.ts`)

```
bridge: id_verification → questionnaire → tos → proof_of_address (optional)
paytrie: id_verification only
```

**Seismic migration:** Replace `bridge` partner with `seismic` partner steps, or collapse to hosted KYC single step.

---

## 11. Database Schema Design

### 11.1 New models (proposed)

```prisma
model SeismicCustomer {
  id                String   @id @default(cuid())
  userId            String   @unique
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  seismicCustomerId String   @unique          // Seismic UUID
  customerType      String   @default("individual") // individual | business
  kycStatus         String   @default("not_started")
  lifecycleStatus   String   @default("PENDING_KYC")
  // PENDING_KYC | KYC_APPROVED | PROVISIONING_ACCOUNTS | ACTIVE | FAILED
  kycLinkUrl        String?
  kycLinkToken      String?
  profileSnapshot   Json?
  referenceId       String?  @unique          // Anzo userId for idempotency if supported

  // Link to legacy during migration
  bridgeCustomerId  String?  @unique
  bridgeCustomer    BridgeCustomer? @relation(fields: [bridgeCustomerId], references: [id])

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  accounts          SeismicAccount[]
  accountRequests   SeismicAccountCreationRequest[]
  orders            SeismicOrder[]
  activityEvents    SeismicActivityEvent[]

  @@index([seismicCustomerId])
  @@index([kycStatus])
  @@index([lifecycleStatus])
}

model SeismicAccountCreationRequest {
  id                  String   @id @default(cuid())
  seismicCustomerId   String
  seismicCustomer     SeismicCustomer @relation(fields: [seismicCustomerId], references: [id], onDelete: Cascade)
  userId              String

  creationRequestId   String   @unique   // From Seismic API
  idempotencyKey      String   @unique   // Anzo-side key used for the POST
  kind                String   // fiat | crypto
  currency            String
  purpose             String   // onramp_deposit | wallet_destination
  status              String   @default("pending") // pending | completed | failed
  seismicAccountId    String?             // Set when completed
  error               Json?

  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  @@index([seismicCustomerId])
  @@index([status])
}

model SeismicAccount {
  id               String   @id @default(cuid())
  seismicCustomerId String
  seismicCustomer  SeismicCustomer @relation(fields: [seismicCustomerId], references: [id], onDelete: Cascade)
  userId           String

  seismicAccountId String   @unique
  kind             String   // fiat | crypto
  shape            String?  // us, iban, evm, uk, mx_clabe, br_pix, co_bank
  currency         String
  rails            String[]
  purpose          String   // onramp_deposit | wallet_destination | payout_destination
  status           String   // active | pending | closed
  details          Json?    // rail-specific deposit instructions or masked bank info

  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  @@unique([seismicCustomerId, currency, purpose]) // enforce 1 USD VA, 1 EUR VA, etc.
  @@index([userId])
}

model SeismicOrder {
  id               String   @id @default(cuid())
  userId           String
  seismicCustomerId String
  seismicCustomer  SeismicCustomer @relation(fields: [seismicCustomerId], references: [id], onDelete: Cascade)

  seismicOrderId   String   @unique
  quoteSnapshotId  String?
  quoteId          String?
  type             String   // ONRAMP | OFFRAMP | SWAP
  status           String   // awaiting_funds | pending | processing | completed | failed | canceled

  sourceCurrency   String
  sourceAmount     String
  sourceRail       String?
  sourceAccountId  String?

  destinationCurrency String
  destinationAmount   String?
  destinationRail     String?
  destinationAccountId String?

  fees             Json?
  rate             String?
  fundingInstructions Json?
  metadata         Json?
  idempotencyKey   String?

  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt
  completedAt      DateTime?

  @@index([userId])
  @@index([status])
  @@index([type])
}

model SeismicActivityEvent {
  id         String   @id @default(cuid())
  userId     String
  externalId String   @unique
  type       String   // SEISMIC_KYC_APPROVED, SEISMIC_DEPOSIT_IN, SEISMIC_PAYOUT_OUT, etc.
  status     String   @default("COMPLETED")
  title      String?
  subtitle   String?
  sourceId   String?
  metadata   Json?
  occurredAt DateTime @default(now())
  createdAt  DateTime @default(now())

  @@index([userId])
}
```

### 11.2 User model addition

```prisma
model User {
  // ... existing ...
  seismicCustomer   SeismicCustomer?
}
```

### 11.3 Legacy tables during migration

**Do not drop** `BridgeCustomer`, `BridgeVirtualAccount`, `KycProfile`, `KycVerification` until all active users migrated. Add `SeismicCustomer.bridgeCustomerId` for correlation.

### 11.4 Field mapping: KycProfile → Seismic KycProfile

| Anzo `KycProfile` | Seismic `profile` path |
|-------------------|------------------------|
| `firstName` | `identity.first_name` |
| `lastName` | `identity.last_name` |
| `dateOfBirth` | `identity.date_of_birth` (YYYY-MM-DD) |
| `nationality` | `identity.nationality` (ISO alpha-3) |
| `phone` | `contact.phone` (E.164) |
| `streetLine1/2, city, state, postalCode, country` | `addresses.residential.*` |
| `occupation`, `employmentStatus` | `employment.*` |
| `sourceOfFunds`, `accountPurpose` | `risk.*` |
| `expectedMonthlyVolume` | `volumes.expected_monthly_volume` |
| `taxId`, `taxIdCountry` | `jurisdictions.tax_id`, `tax_id_country` |
| `signedAgreementId` (Bridge TOS) | `attestations.terms_accepted` |

**Country codes:** Seismic native uses **ISO alpha-3** in profiles; Bridge compat uses **alpha-2** in addresses — normalize in mapper.

---

## 12. Backend Module Design

### 12.1 Proposed file structure

```
src/modules/seismic/
├── seismic.module.ts
├── seismic.controller.ts              # User-facing REST (thin)
├── seismic-webhook.controller.ts      # POST /api/v1/webhooks/seismic
├── seismic.service.ts                 # Facade
├── seismic.provider.ts                # SDK wrapper (SeismicOrchestrationClient)
├── providers/
│   └── seismic-bridge-compat.provider.ts   # Optional: compat client for transition
├── handlers/
│   ├── seismic-kyc.handler.ts         # Customer create, KYC status, approval side-effects
│   ├── seismic-account.handler.ts     # VA provisioning, external account CRUD
│   ├── seismic-quote.handler.ts       # Quote create, accept, poll
│   ├── seismic-payout.handler.ts      # Off-ramp orchestration (replaces BridgePayoutHandler for Seismic users)
│   ├── seismic-onramp.handler.ts      # Deposit instructions read API
│   ├── seismic-webhook.handler.ts     # Event routing
│   └── seismic-activity.handler.ts    # Activity feed parity
├── dto/
│   ├── create-customer.dto.ts
│   ├── create-quote.dto.ts
│   ├── create-external-account.dto.ts
│   └── virtual-account-response.dto.ts
└── utils/
    ├── profile-mapper.util.ts         # KycProfile ↔ Seismic profile
    ├── status-mapper.util.ts          # Order/KYC status mapping
    ├── country-code.util.ts           # alpha-2 ↔ alpha-3
    └── seismic-error.util.ts          # SeismicError → HTTP exceptions
```

### 12.2 Handler responsibilities

| Handler | Replaces (for Seismic cohort) | Key methods |
|---------|------------------------------|-------------|
| `SeismicKycHandler` | `BridgeKycHandler` + parts of `PartnerSubmissionHandler` | `createCustomer`, `onKycApproved`, `tryActivateCustomer`, `getStatus` |
| `SeismicAccountHandler` | VA creation in `handleKycApproval` | `provisionDefaultAccounts`, `registerTurnkeyWallet`, `onAccountCreated`, `addFiatExternal` |
| `SeismicQuoteHandler` | Direct Bridge transfer creation | `createOnrampQuote` (**with `named: "require"`**), `createOfframpQuote`, `acceptBest`, `getSnapshot` |
| `SeismicPayoutHandler` | `BridgePayoutHandler` | `quotePayout`, `prepare`, `execute`, `getStatus` |
| `SeismicOnrampHandler` | `BridgeOnrampHandler.getVirtualAccount` | `getVirtualAccounts` |
| `SeismicWebhookHandler` | `BridgeWebhookController` + Sumsub webhook (Path A) | `handleEvent` |

### 12.3 Provider selection (feature flag)

```typescript
// Pseudocode — inject based on FF_SEISMIC_USER or user.seismicCustomer presence
interface FiatRailsProvider {
  getVirtualAccounts(userId: string): Promise<VirtualAccountResponse>;
  addExternalAccount(userId: string, dto: CreateExternalAccountDto): Promise<ExternalAccount>;
  quotePayout(userId: string, dto: PayoutQuoteDto): Promise<QuoteSnapshot>;
  preparePayout(userId: string, dto: PreparePayoutDto): Promise<PreparePayoutResponse>;
  executePayout(userId: string, dto: ExecutePayoutDto): Promise<PayoutStatus>;
}
```

### 12.4 `SeismicProvider` (SDK wrapper requirements)

All outbound Seismic calls go through `SeismicProvider`. Responsibilities:

1. **Environment selection** — `SeismicOrchestrationClient.sandbox()` vs `.production()` from `SEISMIC_ENV`
2. **Idempotency injection** — every mutating call uses `mutate()` with deterministic keys ([§7.4.3](#743-idempotency-keys-mandatory))
3. **Error normalization** — map `SeismicError.code` to NestJS exceptions (`QuoteExpired` → 422, `AlreadyAccepted` → treat as success)
4. **No dual credentials** — never send both `Api-Key` and `Authorization: Bearer` (returns `400 BothCredsProvided`)

```typescript
@Injectable()
export class SeismicProvider {
  private readonly api: ReturnType<typeof SeismicOrchestrationClient.sandbox>;

  constructor(config: ConfigService) {
    const apiKey = config.getOrThrow('SEISMIC_API_KEY');
    const env = config.get('SEISMIC_ENV', 'sandbox');
    this.api = env === 'production'
      ? SeismicOrchestrationClient.production({ apiKey })
      : SeismicOrchestrationClient.sandbox({ apiKey });
  }

  /** All state-mutating calls MUST use this — idempotency is not optional. */
  mutate<T>(options: {
    fn: (idempotencyKey: string) => Promise<T>;
    idempotencyKey: string;
  }): Promise<T> {
    return options.fn(options.idempotencyKey);
  }

  getApi() {
    return this.api;
  }
}
```

---

## 13. API Surface

### 13.1 New Anzo endpoints (proposed)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/v1/seismic/onboarding/start` | Yes | Create Seismic customer + return KYC link |
| `GET` | `/api/v1/seismic/onboarding/status` | Yes | KYC + lifecycle + account readiness |
| `GET` | `/api/v1/seismic/virtual-accounts` | Yes | USD/EUR deposit instructions (Bridge parity) |
| `GET` | `/api/v1/seismic/schemas` | Yes | Payout bank form schemas (can reuse `BridgeSchemaHandler` + add COP) |
| `POST` | `/api/v1/seismic/external-accounts` | Yes | Register payout bank |
| `GET` | `/api/v1/seismic/external-accounts` | Yes | List payout banks |
| `DELETE` | `/api/v1/seismic/external-accounts/:id` | Yes | Deactivate |
| `POST` | `/api/v1/seismic/payout/quote` | Yes | Quote off-ramp options |
| `POST` | `/api/v1/seismic/payout/prepare` | Yes | Prepare funding steps |
| `POST` | `/api/v1/seismic/payout/execute` | Yes | Accept quote / submit order |
| `GET` | `/api/v1/seismic/payout/:orderId/status` | Yes | Poll order status |
| `GET` | `/api/v1/seismic/orders` | Yes | Transaction history |
| `POST` | `/api/v1/webhooks/seismic` | No (Svix sig) | Unified webhook |

### 13.2 Existing endpoints — migration mapping

| Current endpoint | Seismic-era behavior |
|------------------|---------------------|
| `GET /api/v1/kyc/status/bridge` | Delegate to `/seismic/onboarding/status` for Seismic users; Bridge fallback for legacy |
| `GET /api/v1/bridge/virtual-account` | Route to Seismic or Bridge provider |
| `POST /api/v1/bridge/payout/*` | Route to Seismic payout handler for Seismic users |
| `POST /api/v1/kyc/webhook/sumsub` | Keep if Path B; deprecate if Path A |
| `POST /api/v1/bridge/webhook` | Legacy cohort only |

### 13.3 Seismic API calls (backend → Seismic)

| Operation | Seismic endpoint | Idempotency-Key (mandatory) |
|-----------|------------------|----------------------------|
| Create customer | `POST /v0/customers` | `customer-{anzoUserId}` |
| Update profile | `PUT /v0/customers/:id/kyc/profile` | — (safe to retry; not idempotent-required) |
| Submit KYC | `POST /v0/customers/:id/kyc/submit` | `kyc-submit-{seismicCustomerId}` |
| USD VA | `POST /v0/customers/:id/accounts/fiat/virtual` | `va-usd-{seismicCustomerId}` |
| EUR VA | `POST /v0/customers/:id/accounts/fiat/virtual` | `va-eur-{seismicCustomerId}` |
| Turnkey register | `POST /v0/customers/:id/accounts/crypto/external` | `wallet-base-{seismicCustomerId}` |
| Fiat external | `POST /v0/customers/:id/accounts/fiat/external` | `ext-{seismicCustomerId}-{hash}` |
| On-ramp quote | `POST /v0/quotes` | `quote-{clientRequestId}` — **include `named: "require"`** |
| Off-ramp quote | `POST /v0/quotes` | `quote-{clientRequestId}` — no `named` flag |
| Accept best | `POST /v0/quotes/:id/accept-best` | `accept-best-{snapshotId}` |
| Accept specific | `POST /v0/quotes/:id/accept` | `accept-{snapshotId}-{quoteId}` |
| Poll creation | `GET /v0/customers/:id/accounts/requests/:request_id` | — |

---

## 14. Webhooks & Event Processing

### 14.1 Seismic → Anzo event mapping

| Seismic event | Anzo action |
|---------------|-------------|
| `customer.kyc.status_changed` → `approved` | Set `lifecycleStatus = PROVISIONING_ACCOUNTS`; kick off `provisionDefaultAccounts()`; push `SEISMIC_KYC_APPROVED` — **do not set ACTIVE yet** |
| `customer.kyc.status_changed` → `rejected` | Set failed/retry state; notify user |
| `account.created` | Upsert `SeismicAccount`; mark `SeismicAccountCreationRequest` completed; call `tryActivateCustomer()` |
| `account.updated` | Sync status/details on existing `SeismicAccount` |
| `order.status_changed` → `completed` | Update `SeismicOrder`; off-ramp → `SEISMIC_PAYOUT_OUT` activity + push |
| `order.status_changed` → `awaiting_funds` | Store `fundingInstructions` on order; surface in prepare response |
| `order.status_changed` → `failed` | Mark failed; notify user |

When `tryActivateCustomer()` succeeds (USD + EUR VA + Turnkey external all `active`):

- Set `lifecycleStatus = ACTIVE`
- Push: "Your bank accounts are ready"
- Activity: `SEISMIC_BANK_ACCOUNTS_READY`

### 14.2 Webhook handler requirements

- Verify with **Svix** (`svix` npm package) using `SEISMIC_WEBHOOK_SECRET`
- Return **200 within 15 seconds** (Svix hard limit — retries on timeout)
- **Process async** via in-process queue or job runner (match Bridge webhook pattern)
- Deduplicate by Svix message ID → `SeismicActivityEvent.externalId` or dedicated `WebhookEvent` table
- **Idempotent handlers:** KYC approved handler must tolerate Svix retries (rely on `SeismicProvider` idempotency keys + DB unique constraints)
- If inline snapshot missing (historical replay), fallback `GET /customers/:id` or `GET /accounts/:id`

### 14.3 Activity feed parity

| Bridge event type | Seismic equivalent |
|-------------------|-------------------|
| `BRIDGE_KYC_APPROVED` | `SEISMIC_KYC_APPROVED` |
| `BRIDGE_BANK_ACCOUNTS_READY` | `SEISMIC_BANK_ACCOUNTS_READY` |
| `BRIDGE_TRANSFER_IN` | `SEISMIC_DEPOSIT_IN` (from completed on-ramp order) |
| `BRIDGE_TRANSFER_OUT` | `SEISMIC_PAYOUT_OUT` |

Wire into existing `transaction-history.handler.ts` with provider discriminator.

---

## 15. Environment Variables & Configuration

### 15.1 New variables (add to `validation.schema.ts` + `.env.example`)

```bash
# Seismic Orchestration
SEISMIC_API_KEY=seismic_sandbox_...                    # Org-scoped API key
SEISMIC_BASE_URL=https://orchestration-sandbox.seismictest.net/api
SEISMIC_WEBHOOK_SECRET=whsec_...                       # From Svix portal
SEISMIC_ENV=sandbox                                    # sandbox | production

# Feature flags
FF_SEISMIC_ONBOARDING=false                            # New users → Seismic
FF_SEISMIC_PAYOUTS=false                               # Payouts via Seismic
FF_SEISMIC_WEBHOOKS=true                                 # Process Seismic webhooks
FF_LEGACY_BRIDGE=true                                  # Keep Bridge for unmigrated users
FF_LEGACY_SUMSUB=true                                  # Keep Sumsub (Path B)
```

### 15.2 Joi validation (proposed)

```typescript
SEISMIC_API_KEY: Joi.string().when('FF_SEISMIC_ONBOARDING', {
  is: 'true',
  then: Joi.required(),
  otherwise: Joi.optional(),
}),
SEISMIC_BASE_URL: Joi.string().uri().default(
  'https://orchestration-sandbox.seismictest.net/api',
),
SEISMIC_WEBHOOK_SECRET: Joi.string().optional(),
SEISMIC_ENV: Joi.string().valid('sandbox', 'production').default('sandbox'),
FF_SEISMIC_ONBOARDING: Joi.string().valid('true', 'false').default('false'),
FF_SEISMIC_PAYOUTS: Joi.string().valid('true', 'false').default('false'),
FF_LEGACY_BRIDGE: Joi.string().valid('true', 'false').default('true'),
```

### 15.3 Prerequisites (non-env)

| Prerequisite | Owner | Blocking |
|--------------|-------|----------|
| Seismic org created + MFA | Ops | Yes |
| Org KYB submitted + **approved** | Ops/Legal | Yes — `KybNotApproved` blocks quoting |
| API key minted (sandbox) | Eng | Yes |
| Svix webhook endpoint configured | Eng | Yes |
| Production API + KYB | Ops | Yes for prod cutover |

---

## 16. Migration Strategy & Feature Flags

### 16.1 Phases

```
Phase 0 — Prerequisites (Ops)
  Seismic dashboard signup, KYB, API keys, webhook portal

Phase 1 — Foundation (Eng, no user impact)
  SeismicModule, schema migration, provider, webhook handler (shadow mode)

Phase 2 — New user cohort
  FF_SEISMIC_ONBOARDING=true for new registrations
  Dual status APIs; monitor webhooks

Phase 3 — Payouts
  FF_SEISMIC_PAYOUTS=true for Seismic cohort
  Validate quote → order → bank delivery in sandbox

Phase 4 — Existing user migration
  ACTIVE Bridge users: re-KYC (Option A) or grandfather (Option C)

Phase 5 — Decommission
  FF_LEGACY_BRIDGE=false, remove Sumsub if Path A
```

### 16.2 Existing user options

| Option | Description | Risk | Effort |
|--------|-------------|------|--------|
| **A — Re-KYC** | Create Seismic customer; user completes hosted KYC | Low data integrity risk | Medium UX friction |
| **B — Data import** | Pre-fill + submit if Seismic supports import | Needs Seismic confirmation | Low UX if supported |
| **C — Grandfather** | Keep Bridge for ACTIVE users indefinitely | Dual integration cost | Low short-term |

**Recommendation:** Option **C** for ACTIVE users initially; Option **A** for in-progress KYC; all **new** users on Seismic.

### 16.3 Rollback

If critical failure: set `FF_SEISMIC_ONBOARDING=false`, route new users back to Bridge+Sumsub; Seismic partial users remain on Seismic until manually resolved.

---

## 17. End-to-End Example: New User Workflow

**Persona:** Maria, US resident, new Anzo user, wants USD bank deposits and EUR payouts.

### Timeline

```
Day 0 — Registration (unchanged)
─────────────────────────────────
Maria signs up via passkey
→ Auth creates User + Turnkey Wallet
→ ethereumAddress = 0xABC... on Base
→ No fiat features yet

Day 0 — Start fiat onboarding (FF_SEISMIC_ONBOARDING=true)
─────────────────────────────────
Maria taps "Add money via bank"

Mobile → POST /api/v1/seismic/onboarding/start

Backend:
  1. seismicApi.customers.create({
       type: "individual",
       profile: { contact: { email: maria@... } }
     })
  2. INSERT SeismicCustomer { kycStatus: "not_started", lifecycleStatus: "PENDING_KYC" }
  3. Return { kycLinkUrl: "https://..." }

Mobile opens kycLinkUrl in WebView

Day 0 — KYC completion
─────────────────────────────────
Maria uploads passport, selfie, fills questionnaire, accepts TOS (all in Seismic UI)

Seismic reviews → status: approved

Webhook → POST /api/v1/webhooks/seismic
  event: customer.kyc.status_changed
  data: { customer_id, status: "approved", kyc_profile: {...} }

Backend SeismicKycHandler.onApproved(mariaUserId):
  1. UPDATE SeismicCustomer SET kycStatus=approved, lifecycleStatus=PROVISIONING_ACCOUNTS
  2. SeismicAccountHandler.provisionDefaultAccounts() — with idempotency keys:
     a. POST .../accounts/fiat/virtual (USD) → AccountCreationRequest (pending)
     b. POST .../accounts/fiat/virtual (EUR) → AccountCreationRequest (pending)
     c. POST .../accounts/crypto/external (Turnkey 0xABC...) 
     d. INSERT 3 SeismicAccountCreationRequest rows
  3. Push: SEISMIC_KYC_APPROVED ("Verification complete — setting up accounts...")
  — Maria sees "Processing..." in app (lifecycleStatus=PROVISIONING_ACCOUNTS)

Webhooks arrive (order may vary):
  account.created (USD VA) → upsert SeismicAccount, tryActivateCustomer()
  account.created (EUR VA) → upsert SeismicAccount, tryActivateCustomer()
  account.created (crypto external) → upsert SeismicAccount, tryActivateCustomer()

When all 3 accounts status=active:
  4. tryActivateCustomer() → lifecycleStatus = ACTIVE
  5. Push: "Your bank accounts are ready"
  6. Activity: SEISMIC_BANK_ACCOUNTS_READY

Day 1 — View deposit instructions
─────────────────────────────────
Maria → GET /api/v1/seismic/virtual-accounts

Response (Bridge-parity shape):
{
  "accounts": [
    {
      "currency": "usd",
      "depositInstructions": { "routingNumber": "...", "accountNumber": "...", "paymentRails": ["ach_push","wire"] },
      "destination": { "currency": "usdc", "paymentRail": "base", "address": "0xABC..." }
    },
    {
      "currency": "eur",
      "depositInstructions": { "iban": "...", "bic": "..." },
      "destination": { "currency": "usdc", "paymentRail": "base", "address": "0xABC..." }
    }
  ]
}

Day 2 — USD on-ramp
─────────────────────────────────
Maria ACHs $500 to USD virtual account

Seismic processes fiat → USDC → sends to 0xABC... on Base

Webhook: order.status_changed → completed (on-ramp order)

Backend:
  - Activity: SEISMIC_DEPOSIT_IN ($500)
  - Maria sees USDC balance in app (existing balance module)

Day 5 — Add EUR payout bank
─────────────────────────────────
Maria → POST /api/v1/seismic/external-accounts
  { currency: "EUR", iban: "DE...", bic: "..." }

Backend → POST .../accounts/fiat/external
INSERT SeismicAccount { purpose: payout_destination, currency: EUR }

Day 5 — Payout €200
─────────────────────────────────
Maria → POST /api/v1/seismic/payout/quote
  { amount: "200", destinationCurrency: "EUR" }

Backend:
  1. POST /v0/quotes {
       end_customer_id: mariaSeismicId,
       amount: "200",
       source: { account: usdcCryptoExternalId, rails: ["base"] },
       destination: { account: eurFiatExternalId, rails: ["sepa"] }
     }
  2. Return ranked quotes to mobile

Maria confirms → POST /api/v1/seismic/payout/execute { snapshotId }

Backend:
  1. POST /v0/quotes/:id/accept-best
  2. INSERT SeismicOrder { status: "awaiting_funds", fundingInstructions: {...} }
  3. If funding requires on-chain send:
     - Relay quote: 0xABC → deposit address from funding_instructions
     - Mobile signs via Turnkey (same UX as today's seamless payout)
  4. Webhooks drive: awaiting_funds → pending → processing → completed
  5. Activity: SEISMIC_PAYOUT_OUT; Push: "€200 sent to your bank"

Day 7 — Maria's account inventory (final)
─────────────────────────────────
SeismicCustomer: ACTIVE, kyc approved
SeismicAccounts:
  [1] USD fiat virtual     — onramp_deposit
  [2] EUR fiat virtual     — onramp_deposit
  [3] USDC crypto external — wallet_destination (0xABC...)
  [4] EUR fiat external    — payout_destination
SeismicOrders: 2 (1 on-ramp deposit, 1 off-ramp payout)
```

---

## 18. Risks, Blockers & Open Questions

### 18.1 Blockers (must resolve before prod)

| # | Blocker | Owner | Mitigation |
|---|---------|-------|------------|
| B1 | Seismic production API not live | Seismic | UAT on sandbox; contractual prod date |
| B2 | Org KYB approval | Anzo ops | Start KYB immediately |
| B3 | Off-ramp quote with Turnkey as source (no custodial wallet) | Eng + Seismic | Sandbox spike in Phase 1 |
| B4 | Seamless payout UX parity without `bridge_wallet` | Eng | Redesign around orders + Relay to `funding_instructions` |

### 18.2 Open questions for Seismic team

1. Can existing Bridge KYC-approved users be imported without re-verification?
2. Does `POST /customers` support `reference_id` = partner user ID for idempotency?
3. For on-ramp VA deposits, is a separate order created or only `account.updated` + balance events?
4. Exact list of KYC-accepted countries for US/EU individual customers under our org KYB?
5. Production go-live date and SLA for webhooks?
6. Developer fee model equivalent to `BRIDGE_DEVELOPER_FEE_PERCENT` / `BRIDGE_TRANSFER_DEVELOPER_FEE`?

### 18.3 Anzo internal decisions

| Decision | Options | Recommendation |
|----------|---------|----------------|
| KYC path | Hosted (A) vs Sumsub retained (B) | **B for MVP** (less mobile churn), migrate to A later |
| Existing ACTIVE users | Re-KYC vs grandfather | **Grandfather** initially |
| Colombia COP | Enable or ignore | Ignore until product requests |
| Seismic crypto virtual | Use or skip | **Skip** — use Turnkey external |

---

## 19. Implementation Phases

| Phase | Duration | Deliverables | Exit criteria |
|-------|----------|--------------|---------------|
| **0 — Ops setup** | 3–5 days | Org, KYB submitted, sandbox API key, Svix endpoint | KYB approved (or sandbox unblocked) |
| **1 — Foundation** | 5–7 days | Prisma models (incl. `SeismicAccountCreationRequest`), `SeismicModule`, `SeismicProvider` with **mandatory idempotency**, webhook handler (async, 200 within 15s) | Customer create + webhook received; idempotency keys verified on retry |
| **2 — Accounts** | 5–7 days | `SeismicAccountHandler` with async lifecycle, `tryActivateCustomer()`, virtual account GET API | USD+EUR VA + Turnkey external reach `active` via `account.created` before `ACTIVE` |
| **3 — Onboarding API** | 3–5 days | `/onboarding/start`, `/status` (handles `PROVISIONING_ACCOUNTS`), feature flags | New user completes KYC in sandbox; gating correct |
| **4 — Payouts** | 7–10 days | `SeismicQuoteHandler` with **`named: "require"`** on on-ramp; off-ramp quote/accept; order tracking | Sandbox off-ramp €/USD successful |
| **5 — Migration** | 7–14 days | Dual routing, legacy fallback, monitoring | Cohort rollout 5% → 100% new users |
| **6 — Cleanup** | 3–5 days | Deprecate Bridge for new users, docs, runbooks | FF_LEGACY_BRIDGE off for new registrations |

**Phase 1 non-negotiables:** R1 (`named: "require"`), R2 (async account creation), R3 (idempotency keys) — see [§7.4](#74-critical-api-behaviors-mandatory--live-docs).

**Total estimate:** ~6–8 weeks with testing and staged rollout.

---

## 20. Appendix: Local Docs Corrections

When using `docs/seismic-documentation/` as reference, apply these corrections:

1. Replace all **"Enso"** → **"Anzo"**
2. Default account provisioning must include **EUR fiat virtual**, not only USD
3. Prefer **crypto external (Turnkey)** over crypto virtual for wallet destination
4. Do not assume Sumsub removal in Phase 1 — Anzo mobile app depends on it today
5. Bridge compat is a **transition tool**, not the target architecture
6. Add **COP** as optional future payout currency not present in Anzo today
7. Live docs verified June 2026 — local mirror is supplementary; prefer live docs for API behavior
8. **`named: "require"`** mandatory on on-ramp quotes (live docs)
9. **`AccountCreationRequest`** async lifecycle — never set ACTIVE synchronously on KYC approve
10. **`Idempotency-Key`** mandatory on all mutating calls via `SeismicProvider`

---

## Related documents

| Document | Role |
|----------|------|
| `docs/VIRTUAL_ACCOUNTS_AND_PAYOUTS.md` | Current Bridge+Sumsub baseline (accurate) |
| `docs/seismic-documentation/01-overview-getting-started.md` | Seismic product overview |
| `docs/seismic-documentation/04-api-reference-customers-accounts.md` | Customers & accounts API |
| `docs/seismic-documentation/05-api-reference-quotes.md` | Quotes & orders |
| `docs/seismic-documentation/07-webhooks.md` | Webhook catalog |
| `docs/seismic-documentation/08-bridge-compat-layer.md` | Compat limitations (critical) |
| `docs/seismic-documentation/seismic-integration-workflow.md` | Draft workflow (needs corrections above) |
| `docs/KYC_INTEGRATION_GUIDE.md` | Mobile KYC contract to update post-decision |

---

**Status: APPROVED FOR IMPLEMENTATION.**

Next step: ticket breakdown per [§19 Implementation Phases](#19-implementation-phases), starting Phase 0 (Ops) + Phase 1 (Foundation with §7.4 mandatory behaviors).
