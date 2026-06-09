# Seismic Orchestration — Overview & Getting Started

> **Full Documentation Analysis** | **For:** Enso Backend Integration Team  
> **Source:** https://orchestration.seismic.systems/docs/  
> **Analyzed:** June 9, 2026

---

## 1. What is Seismic Orchestration?

Seismic Orchestration is a **REST API + TypeScript client** for on/off-ramp, KYC, stablecoin swaps, and wallets. It provides a unified platform for fintechs to onboard end-users, verify their identity (KYC/KYB), create accounts, request quotes across multiple payment rails, and execute transfers.

### Key Capabilities
| Capability | Description |
|-----------|-------------|
| **On/Off-Ramp** | Convert fiat ↔ crypto across multiple rails (ACH, Wire, SEPA, PIX, Base, Ethereum, Solana, etc.) |
| **KYC/KYB** | Hosted identity verification for individuals (KYC) and businesses (KYB) |
| **Stablecoin Swaps** | Cross-chain stablecoin transfers |
| **Wallets** | Virtual and external account management (fiat + crypto) |
| **Quotes** | Fan-out pricing across all eligible rails with best-quote selection |

---

## 2. Architecture Overview

### Two Audiences in One API
The Seismic API serves two distinct audiences that never overlap:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEISMIC ORCHESTRATION API                     │
├─────────────────────────────┬───────────────────────────────────┤
│   YOU (Integrating Partner) │   END-CUSTOMERS (Your Users)      │
│                             │                                    │
│  • Dashboard logins         │  • Individuals/Businesses          │
│  • Org-scoped API keys      │  • Created via YOUR app            │
│  • Your org's KYB           │  • KYC/KYB driven by YOU           │
│  • Membership management    │  • Accounts attached per-customer  │
│                             │  • Quotes/Orders on their behalf   │
├─────────────────────────────┼───────────────────────────────────┤
│  Endpoints: /orgs/*         │  Endpoints: /customers/*           │
│            /auth/*          │           /accounts/*              │
│                             │           /quotes/*                │
└─────────────────────────────┴───────────────────────────────────┘
```

### Four Identity Tiers

| Tier | Who | Table | Verification |
|------|-----|-------|-------------|
| **1** | Seismic (the platform itself) | — | — |
| **2** | **Org** — a fintech that signs up | `orgs` | KYB |
| **3** | **Org Admin** — human with dashboard credentials | `users` + `org_memberships` | None directly (personal info travels in org's KYB as `associated_persons[]`) |
| **4** | **End-Customer** — individual/business that the org onboards | `customer_individuals` / `customer_businesses` | KYC (individuals) or KYB (businesses) |

### Getting tiers confused is the #1 cause of "wrong endpoint" bugs.

---

## 3. Base URLs

| Environment | Base URL | Status |
|------------|----------|--------|
| **Sandbox** | `https://orchestration-sandbox.seismictest.net/api` | Active |
| **Production** | Coming soon | Not yet available |

All endpoints live under `/api/v0/*` — paths in per-endpoint docs drop the `/v0/` prefix for readability.

### Bridge Compat Base URLs
| Environment | Base URL |
|------------|----------|
| **Sandbox** | `https://orchestration-sandbox.seismictest.net/api/compat/bridge/v0` |
| **Production** | Coming soon |

---

## 4. Getting Started — Step by Step

### Step 1: Sign up + Create an Org
1. Go to **dashboard.seismic.systems/signup**
2. Create individual account with email + password
3. Enable MFA (required)
4. Create an org with display name (e.g., "Acme Inc.")
5. The signup user becomes the org's **owner**

### Step 2: Submit KYB (Know Your Business)
Fill out the unified KYB profile for your org in the dashboard:

| Section | Fields Collected |
|---------|-----------------|
| **Business Identity** | Legal name, entity type, tax ID, formation date |
| **Addresses** | Registered + optional physical address |
| **Contact** | Primary contact info |
| **Operations** | Jurisdictions, source of funds, regulated status |
| **Volumes** | Monthly transaction buckets |
| **Counterparties** | Expected transaction counterparties |
| **Associated Persons** | UBOs, control persons, signers |

- Save partial progress at any time (dashboard keeps drafts client-side)
- Submit triggers: `PUT /org/kyb/profile` → `POST /org/kyb/submit`
- Org enters `submitted` state
- Quoting and end-user onboarding unlock once org is **approved**

### Step 3: Mint an API Key
1. Dashboard: Settings → API keys → **New key**
2. Label it (e.g., `ci-pipeline`)
3. Copy the secret — **shown once only**
4. Environment is implicit: the dashboard you're logged into determines sandbox vs production

**For programmatic minting:** `POST /auth/api-keys`

### Step 4: Install the SDK

```bash
# npm
npm install seismic-orchestration

# bun
bun add seismic-orchestration

# pnpm
pnpm add seismic-orchestration

# yarn
yarn add seismic-orchestration
```

- Single runtime dependency: `zod`
- Runs in browsers, Node ≥ 18, Bun, Deno, and service workers

### Step 5: Create a Client & Make a Call

```typescript
import { SeismicOrchestrationClient } from "seismic-orchestration";

// Sandbox client
const api = SeismicOrchestrationClient.sandbox({
  apiKey: "seismic_sandbox_...",  // Server-to-server
});

// Or with access token (dashboard sessions)
const api = SeismicOrchestrationClient.sandbox({
  tokenStore: { ... },  // For browser/dashboard auth
});
```

---

## 5. Package Structure

```
seismic-orchestration (npm package)
├── SeismicOrchestrationClient    # Main client class
│   ├── .sandbox()                # Create sandbox client
│   └── .production()             # Create production client
│
├── API Namespaces:
│   ├── api.orgs.*                # Org management, KYB
│   ├── api.auth.*                # Signup, login, MFA, API keys
│   ├── api.customers.*           # Customer CRUD, KYC, accounts
│   ├── api.accounts.*            # Account listing, reading
│   ├── api.quotes.*              # Quote creation, acceptance
│   ├── api.rates.*               # Reference rates
│   ├── api.rails.*               # Supported rails
│   ├── api.schemas.*             # Rail schemas
│   └── api.currencies.*          # Supported currencies
│
└── Features:
    ├── Single-flight refresh-on-401
    ├── Runtime response validation (Zod)
    └── Normalized request shapes
```

---

## 6. Key Concepts

### Orgs (Tenants)
- The **org** is the tenant. All resources (customers, API keys, KYB, quotes, orders) are scoped to an org
- A user can belong to multiple orgs ("Switch org" UI on dashboard)
- Multi-org API callers mint a **separate key per org**
- API keys are **permanent** and don't switch context

### Customers (End-Users)
- Individuals or businesses that YOUR product onboards
- Scoped to your org — they never log into Seismic
- You drive their KYC and attach their funding/payout accounts
- **Polymorphic**: `type: "individual"` (KycProfile) or `type: "business"` (KybProfile)

### Accounts
- First-party: every account belongs to exactly one customer
- The customer IS the beneficiary — no separate beneficiary field
- Types: fiat (virtual/external) + crypto (virtual/external)
- Virtual = Seismic-managed; External = you register existing bank/wallet

### Quotes
- **Ranked comparison** across every eligible rail for a given (source → destination, amount)
- Response contains: snapshot id, list of quotes, pre-computed `best_quote_id`
- **Fan-out**: Omit `rails` on source to get pricing across ALL USD-in rails
- Accept a quote to turn it into an **order**

### Orders
- Created when a quote is accepted
- Lifecycle: `pending` → `submitted` → `settled` / `failed`
- Idempotent — safe to retry accept calls

---

## 7. What the SDK Provides

1. **Single-flight refresh-on-401**: If 50 concurrent requests 401, the client refreshes **once**, not 50 times
2. **Runtime response validation**: All responses validated against Zod schemas mirrored from server DTOs
3. **Normalized request shapes**: `customers.create`, `createAccount`, etc. have clean, typed interfaces

---

## 8. Next Steps After Setup

| Step | Action | Documentation |
|------|--------|--------------|
| 1 | Understand authentication flows | `02-authentication.md` |
| 2 | Review Orgs & Auth API | `03-api-reference-orgs-auth.md` |
| 3 | Implement customer onboarding (KYC) | `04-api-reference-customers-accounts.md` |
| 4 | Implement quotes & transfers | `05-api-reference-quotes.md` |
| 5 | Handle webhooks | `07-webhooks.md` |
| 6 | Review Bridge compat layer (if migrating) | `08-bridge-compat-layer.md` |
| 7 | Review integration workflow | `seismic-integration-workflow.md` |

---

## 9. Important Notes for Enso Integration

### For Enso's Use Case (Neobank Model)
> Acme Inc. (Enso) signs up to use Seismic to move money for its app's users. Enso's admin is a Tier-3 user; Enso is Tier 2 (drives KYB). Admin mints an API key and the app starts creating one customer per signed-up app user (Tier 4) with `type: "individual"`. Each Tier-4 customer goes through KYC via the polymorphic customer surface.

### Migration from Bridge
- Seismic provides a **Bridge compat layer** at `/api/compat/bridge/v0/*`
- Same API key works for both native and compat surfaces
- A **base-URL swap** gets most Bridge integrations running without code changes
- See `08-bridge-compat-layer.md` for full details

### Critical: Sandbox vs Production
- Sandbox tokens start with `sbx:` (for Sumsub compat) or `seismic_sandbox_` (for Seismic)
- Production tokens do NOT have sandbox prefix
- The `SUMSUB_BASE_URL` stays the same for both environments

---

## 10. Status Page & Resources

- **GitHub:** https://github.com/SeismicSystems/ramps
- **Dashboard:** https://dashboard.seismic.systems
- **Docs:** https://orchestration.seismic.systems/docs/
- **Issues:** https://github.com/SeismicSystems/ramps/issues
