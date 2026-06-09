# Seismic Orchestration — API Reference: Quotes

> **Full Documentation Analysis** | **Source:** https://orchestration.seismic.systems/docs/api/quotes  
> **Analyzed:** June 9, 2026

---

## 1. Quotes Overview

A **quote** is a ranked comparison across every eligible rail for a given (source → destination, amount). The response contains:
- A **snapshot id**
- A list of **quotes** (ranked best-first)
- A pre-computed **`best_quote_id`**

Accept a quote to turn it into an **order**.

---

## 2. Quote Endpoints

### `POST /quotes` — Create a quote request
Fan out across rails. Returns an `AsyncQuoteSnapshot`.

```bash
POST /quotes
Api-Key: seismic_...
Content-Type: application/json

{
  "end_customer_id": "7c4b...",
  "end_customer_type": "individual",
  "amount": "1000",
  "source": {
    "currency": "USD"
    // rails omitted — fans out across every USD-in rail
  },
  "destination": {
    "currency": "USDC",
    "rails": ["base"]
  }
}
```

### `GET /quotes/:id` — Re-read a snapshot
Poll for status updates (especially when using async mode).

### `POST /quotes/:id/accept` — Accept a specific quote
```bash
POST /quotes/:id/accept
{
  "quote_id": "q_9e4d..."
}
```

### `POST /quotes/:id/accept-best` — Accept the best quote
No payload needed — accepts the snapshot's `best_quote_id`.
```bash
POST /quotes/:id/accept-best
{}  // Empty body
```

---

## 3. Sync vs. Async Modes

The `create` method has two modes, controlled by the `wait` option:

### Sync Mode: `wait = true`
```typescript
const quotes = await api.quotes.create(req, { wait: true });
// quotes.status === "ready" — every option has been priced
```
- Blocks until every eligible rail has priced the trade
- Returns `AsyncQuoteSnapshot` with `status: "ready"`
- Use for synchronous flows where user is waiting

### Async Mode: Default (no wait)
```typescript
const snapshot = await api.quotes.create(req);
// snapshot.status === "pending" — pricing in progress
// Poll GET /quotes/:id until status === "ready"
```
- Returns immediately with `status: "pending"`
- Poll `GET /quotes/:id` until `status === "ready"`
- Use for fire-and-forget flows

---

## 4. Directionality

The quote API is **direction-agnostic**: onramp, offramp, and stablecoin swap are all the same shape.

| Scenario | source | destination |
|----------|--------|-------------|
| **Onramp** | fiat (USD via ach/wire, …) | stablecoin (USDC on base, …) |
| **Offramp** | stablecoin | fiat (EUR via sepa, …) |
| **Swap** | stablecoin | stablecoin (different rail) |

---

## 5. Fan-Out and Rails

### Omitting Rails (Fan-Out)
```typescript
source: { currency: "USD" }
// Fans out across every USD-in rail (ACH, Wire, SEPA, etc.)
// Returns one quote per rail
```

### Specifying Rails
```typescript
destination: {
  currency: "USDC",
  rails: ["base"]  // Only price Base chain
}
```

### Rail Examples
| Kind | Rails |
|------|-------|
| **Fiat** | `ach`, `wire`, `swift`, `sepa`, `upi`, `pix`, `faster_payments`, `spei` |
| **Crypto** | `base`, `ethereum`, `polygon`, `solana`, `arbitrum`, `optimism`, `avalanche_c_chain`, `celo`, `stellar`, `tron` |

### Schemas (Rail Groups)
Instead of listing individual rails, use schemas:
- `"us_bank"` → `["ach", "wire", "swift"]`
- `"evm"` → `["base", "ethereum", "polygon", "arbitrum"]`

```typescript
destination: {
  currency: "USDC",
  schemas: ["evm"]  // Price all EVM chains
}
```

---

## 6. Each Side of a Quote

Each side (source/destination) is one of three forms:

### Form 1: Bare account id string
```json
"acc_1a2b..."
```
- Shorthand for an attached Account
- Currency is pinned by the account row
- Only valid when account supports exactly ONE rail
- Returns `AmbiguousRail` error if account has multiple rails

### Form 2: Account with disambiguation
```json
{
  "account": "acc_1a2b...",
  "rails": ["ach"],
  "schemas": ["us_bank"]
}
```
- Use for fiat accounts reachable via multiple rails (e.g., US bank with ACH + wire)

### Form 3: Inline descriptor
```json
{
  "currency": "USD",
  "rails": ["ach"],
  "schemas": ["us_bank"],
  "address": "0x..."
}
```
- Pricing-only quotes (no settlement account yet)
- One-off destinations not attached to the customer

---

## 7. Amounts and Fees — The Invariant

```
input amount = sum of all outputs + sum of all fees
```

### Fee Breakdown
```typescript
{
  fees: {
    items: [
      { type: "flat", amount: "3.00", currency: "USD" },
      { type: "percentage", amount: "0.50", currency: "USD" }
    ],
    total: "3.50",
    currency: "USD"
  }
}
```

### Quote Structure
```typescript
{
  id: "q_7f3a...",
  in: {
    amount: "1000",
    currency: "USD",
    rail: "ach",
    account_id: "acc_8a01..."
  },
  out: {
    amount: "997",
    currency: "USDC",
    rail: "base",
    account_id: "acc_3c92..."
  },
  fees: {
    items: [{ type: "flat", amount: "3.00", currency: "USD" }],
    total: "3.00",
    currency: "USD"
  },
  rate: "0.997",
  expires_at: "2026-01-15T12:00:00Z"
}
```

---

## 8. Quote Types

### QuoteRequest
```typescript
{
  end_customer_id: string;        // UUID
  end_customer_type: "individual" | "business";
  amount: string;                 // Decimal string
  source: LegRequest;
  destination: LegRequest;
}
```

### LegRequest
```typescript
// Form 1: Account reference
string  // "acc_..."

// Form 2: Account with disambiguation
{
  account: string;
  rails?: string[];
  schemas?: string[];
}

// Form 3: Inline descriptor
{
  currency: string;
  rails?: string[];
  schemas?: string[];
  address?: string;   // For one-off crypto destinations
}
```

### AsyncQuoteSnapshot
```typescript
{
  id: string;                     // Snapshot UUID
  status: "pending" | "ready";
  best_quote_id: string;          // Pre-computed best option
  quotes: Quote[];                // Ranked best-first
  created_at: string;
  expires_at: string;             // Global snapshot expiry
}
```

### Quote
```typescript
{
  id: string;                     // Quote ID for acceptance
  in: Leg;                        // Input side
  out: Leg;                       // Output side
  fees: FeeBreakdown;
  rate: string;                   // Exchange rate
  expires_at: string;             // Per-quote expiry
}
```

### Leg
```typescript
{
  amount: string;
  currency: string;
  rail: string;
  account_id?: string;
}
```

---

## 9. Accepting a Quote

### Accept Best (Recommended)
```typescript
const order = await api.quotes.acceptBest(snapshotId);
// No payload — uses snapshot.best_quote_id
```

### Accept Specific
```typescript
const order = await api.quotes.accept(snapshotId, "q_9e4d...");
// Pass specific quote_id from snapshot.quotes[]
```

### Response (Order)
```typescript
{
  id: string;
  status: "awaiting_funds" | "pending" | "processing" | "completed" | "expired" | "failed" | "canceled";
  customer_id: string;
  idempotency_key: string;
  source: { ... };
  destination: { ... };
  funding_instructions?: { ... };  // For awaiting_funds
  created_at: string;
  updated_at: string;
}
```

---

## 10. Idempotency

All state-mutating endpoints accept an **`Idempotency-Key`** header:
```bash
POST /quotes
Idempotency-Key: my-unique-key-123
```
- Same key + same payload → same response (safe to retry)
- Different payload + same key → 409 conflict
- Keys expire after 24 hours

---

## 11. Error Handling

### Quote-Specific Error Codes
| Code | Status | Meaning |
|------|--------|---------|
| `NoEligibleOptions` | 422 | No option priced this corridor |
| `QuoteNotFound` | 404 | Snapshot id unknown (or not yours) |
| `QuoteNotReady` | 422 | Snapshot still pending — poll GET |
| `QuoteExpired` | 422 | Quote's expires_at has passed. Re-create. |
| `QuoteIdInvalid` | 400 | quote_id not in snapshot.quotes[] |
| `AlreadyAccepted` | 409 | Snapshot already turned into order. Treat as success. |
| `AmbiguousRail` | 400 | Account supports multiple rails; none specified |
| `SchemaNotApplicable` | 400 | Schema kind doesn't match leg's account family |
| `SchemaNotFound` | 400 | Unknown schema id |

### Retry Guidance
- `QuoteExpired` → re-create snapshot, show new price, let user re-confirm
- `AlreadyAccepted` → treat as success (prior request won the race)
- `QuoteIdInvalid` → UI bug, log + report
- 5xx → retry with exponential backoff, ~5 attempts max

---

## 12. Enso Integration: Quotes Flow

### Onramp Flow (Fiat → Crypto)
```typescript
// 1. Create quote (fan out across all USD rails)
const snapshot = await api.quotes.create({
  end_customer_id: customerId,
  end_customer_type: "individual",
  amount: "1000",
  source: { currency: "USD" },           // Fan out across all USD rails
  destination: { currency: "USDC", rails: ["base"] }
}, { wait: true });

// 2. Show ranked options to user
for (const q of snapshot.quotes) {
  console.log(`${q.in.rail} → ${q.out.amount} ${q.out.currency} (rate: ${q.rate}, fee: ${q.fees.total})`);
}

// 3. Accept best (or let user pick)
const order = await api.quotes.acceptBest(snapshot.id);

// 4. Show funding instructions
if (order.status === "awaiting_funds") {
  showFundingInstructions(order.funding_instructions);
}
```

### Offramp Flow (Crypto → Fiat)
```typescript
const snapshot = await api.quotes.create({
  end_customer_id: customerId,
  end_customer_type: "individual",
  amount: "500",
  source: { currency: "USDC", rails: ["base"] },
  destination: { currency: "EUR", rails: ["sepa"] }
}, { wait: true });

const order = await api.quotes.acceptBest(snapshot.id);
```

### Swap Flow (Crypto → Crypto)
```typescript
const snapshot = await api.quotes.create({
  end_customer_id: customerId,
  end_customer_type: "individual",
  amount: "100",
  source: { currency: "USDC", rails: ["ethereum"] },
  destination: { currency: "USDC", rails: ["base"] }
}, { wait: true });

const order = await api.quotes.acceptBest(snapshot.id);
```

---

## 13. Comparison: Enso's Current Bridge Integration vs Seismic Quotes

| Aspect | Enso Current (Direct Bridge) | Seismic |
|--------|------------------------------|---------|
| Quote creation | Direct Bridge API | `POST /quotes` with fan-out |
| Rail selection | Manual per-rail calls | Automatic fan-out + ranking |
| Best price | Client-side comparison | Server-computed `best_quote_id` |
| Quote acceptance | Direct Bridge transfer | `POST /quotes/:id/accept` |
| Idempotency | Bridge idempotency key | Seismic `Idempotency-Key` header |
| Status tracking | Poll Bridge transfer | Webhooks (`order.status_changed`) |
