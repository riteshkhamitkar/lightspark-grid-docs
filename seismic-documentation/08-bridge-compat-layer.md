# Seismic Orchestration — Bridge Compatibility Layer

> **Full Documentation Analysis** | **Source:** https://orchestration.seismic.systems/docs/bridge-compat/intro  
> **Analyzed:** June 9, 2026

---

## 1. Overview

Bridge is an independent product; **Seismic Orchestration does not use it under the hood**. The compat layer mirrors Bridge's API shape so teams already integrated with Bridge can **swap base URLs** and keep most code unchanged.

### TL;DR — Base URL Swap
```diff
- https://api.bridge.xyz/v0
+ https://orchestration-sandbox.seismictest.net/api/compat/bridge/v0
```

### Key Points
| Point | Detail |
|-------|--------|
| **Same auth** | One `Api-Key` works for both native and compat surfaces |
| **Same tenant** | Same account, two API shapes |
| **Best-effort** | Not a contract; may have inconsistencies |
| **New integrations** | Use native API (`/api/v0/*`); compat is for existing Bridge callers |
| **501 fallback** | Unimplemented endpoints return `501 not_implemented` |

---

## 2. Base URLs

| Environment | Base URL |
|------------|----------|
| **Sandbox** | `https://orchestration-sandbox.seismictest.net/api/compat/bridge/v0` |
| **Production** | Coming soon |

The trailing `/v0` mirrors Bridge's own path versioning.

---

## 3. Authentication

Bridge-style `Api-Key` header. No `Bearer` prefix, no refresh dance.

```bash
curl https://orchestration.seismic.systems/api/compat/bridge/v0/customers \
  -H "Api-Key: seismic_sandbox_..."
```

**The same `SEISMIC_ORCHESTRATION_API_KEY` works against the native API** — one key, two surfaces.

---

## 4. What's Implemented

### Implemented Endpoints

| Resource | Endpoints | Status |
|----------|-----------|--------|
| **Customers** | `POST /customers`, `GET /customers`, `GET /customers/{id}`, `PUT /customers/{id}`, `DELETE /customers/{id}`, `GET /customers/{id}/transfers`, `GET /customers/{id}/kyc_link` | ✅ Implemented |
| **TOS Links** | `POST /customers/tos_links`, `POST /customers/tos_acceptance_link`, `GET /customers/{id}/tos_acceptance_link` | ✅ Implemented |
| **Associated Persons** | `POST /customers/{id}/associated_persons`, `GET /customers/{id}/associated_persons`, `DELETE /customers/{id}/associated_persons/{person_id}` | ✅ Implemented |
| **KYC Links** | `POST /kyc_links`, `GET /kyc_links`, `GET /kyc_links/{id}` | ✅ Implemented |
| **External Accounts** | `GET /external_accounts`, `GET /external_accounts/{id}`, `PUT /external_accounts/{id}`, `DELETE /external_accounts/{id}`, `POST /external_accounts/{id}/reactivate`, `POST /customers/{id}/external_accounts`, `GET /customers/{id}/external_accounts` | ✅ Implemented |
| **Transfers** | `POST /transfers`, `GET /transfers`, `GET /transfers/{id}`, `PUT /transfers/{id}`, `DELETE /transfers/{id}` | ✅ Implemented |
| **Exchange Rates** | Reference rates | ✅ Implemented |
| **Shared Schemas** | Address, IdentifyingInformation, Document, etc. | ✅ Implemented |

### How Transfers Map
The native API splits pricing and settlement:
- `POST /quotes` (price discovery) + `POST /quotes/:id/accept` (settle)

The compat layer **collapses both**:
- `POST /transfers` → runs fan-out server-side + accepts best result
- Returns single synchronous response in Bridge's shape

---

## 5. Not Implemented

These endpoints return `501 not_implemented`:

### Critical for Enso (May Need Workarounds)
| Endpoint | Impact | Workaround |
|----------|--------|------------|
| **Virtual Accounts** | `GET /customers/{id}/virtual_accounts` | Use native API `POST /customers/:id/accounts/fiat/virtual` |
| **Bridge Wallets** | All `/bridge_wallets/*` | Use native API crypto accounts |
| **Cards** | All `/cards/*` | Not supported — plan alternative |

### Less Critical
| Endpoint | Notes |
|----------|-------|
| `POST /batch_settlements` | Batch operations |
| `POST /liquidation_addresses` | Liquidation flow |
| `GET /lists/*` | Reference lists |
| `POST /plaid/*` | Plaid integration |
| `GET /developers/*` | Developer management |
| `GET /rewards/*` | Rewards program |
| `GET /funds_requests/*` | Funds requests |
| `GET /prefunded_accounts/*` | Prefunded accounts |
| `POST /crypto_return_policies` | Return policies |
| `GET /static_memos/*` | Static memos |
| `POST /transfers/templates` | Transfer templates |
| `GET /customers/{id}/static_templates` | Static templates |
| `GET /fiat_payout_configurations/*` | Payout configuration |

### Full 501 Error Response
```json
{
  "code": "not_implemented",
  "message": "This endpoint is not implemented by the Seismic Bridge compat layer."
}
```

---

## 6. Shared Schemas (Bridge Compat)

### Address
```typescript
{
  street_line_1: string;
  street_line_2?: string;
  city: string;
  state?: string;
  postal_code: string;
  country: string;  // ISO alpha-2 (different from native API!)
}
```

### IdentifyingInformation
```typescript
{
  type: "ssn" | "ein" | "passport" | "drivers_license" | ...;
  number: string;
  issuing_country?: string;
  issued_date?: string;
  expiration_date?: string;
}
```

### Document
```typescript
{
  type: "passport" | "drivers_license" | "utility_bill" | ...;
  file_name: string;
  file_type: string;
  file_size: number;
  url?: string;
}
```

### AssociatedPerson
```typescript
{
  id?: string;
  first_name: string;
  last_name: string;
  role: "ubo" | "control_person" | "signer" | ...;
  ownership_percentage?: number;
  date_of_birth?: string;
  address?: Address;
  identifying_information?: IdentifyingInformation;
}
```

---

## 7. KYC Links (Bridge Compat)

### `POST /kyc_links` — Create hosted KYC + TOS link
- One call creates both the hosted KYC flow AND the stub customer behind it
- Use when you want the customer to do the typing

### `GET /kyc_links` — List all KYC links
### `GET /kyc_links/{id}` — Poll KYC link status

---

## 8. Error Differences (Bridge vs Native)

| Aspect | Bridge | Seismic Compat |
|--------|--------|----------------|
| Error shape | `{ "code": "...", "message": "..." }` | Same shape |
| 401 response | Bridge shape | Mirrors Bridge exactly |
| Missing key | `{ "code": "invalid_api_key" }` | Same shape |

---

## 9. Enso Integration Strategy

### Option A: Native API (Recommended for New Code)
Use Seismic's native API for all new functionality:
- `POST /customers` (native)
- `PUT /customers/:id/kyc/profile` (native)
- `POST /quotes` (native)
- `POST /customers/:id/accounts/*` (native)

### Option B: Bridge Compat (For Minimal Migration)
If Enso wants minimal code changes:

```typescript
// Change base URL only
const seismicBridgeApi = axios.create({
  baseURL: "https://orchestration-sandbox.seismictest.net/api/compat/bridge/v0",
  headers: { "Api-Key": process.env.SEISMIC_API_KEY }
});

// Existing Bridge code works with minimal changes
const customer = await seismicBridgeApi.post("/customers", {
  type: "individual",
  first_name: "Alice",
  last_name: "Liddell",
  // ... same as before
});
```

### Option C: Hybrid (Recommended for Migration)
Use Bridge compat for existing flows, native API for new features:

```typescript
// Existing flows via compat
const bridgeCompat = new BridgeCompatClient({ apiKey: SEISMIC_API_KEY });

// New flows via native
const seismicNative = SeismicOrchestrationClient.sandbox({ apiKey: SEISMIC_API_KEY });

// Legacy: Create customer via Bridge compat
const customer = await bridgeCompat.customers.create({ ... });

// New: Get advanced quotes via native
const quotes = await seismicNative.quotes.create({ ... });
```

---

## 10. Migration Checklist

### Phase 1: Bridge Compat Swap (Minimal Changes)
- [ ] Update base URL from `api.bridge.xyz` to Seismic compat URL
- [ ] Update API key from `BRIDGE_API_KEY` to `SEISMIC_API_KEY`
- [ ] Test all existing flows
- [ ] Handle 501 errors for unimplemented endpoints

### Phase 2: Native API Adoption (Recommended)
- [ ] Migrate customer creation to native API
- [ ] Migrate KYC to native hosted KYC links
- [ ] Migrate quotes to native quote fan-out
- [ ] Migrate account creation to native accounts API
- [ ] Implement Seismic webhooks

### Phase 3: Cleanup
- [ ] Remove Bridge SDK dependency
- [ ] Remove Bridge-specific code
- [ ] Remove Sumsub integration (if using Seismic KYC)
- [ ] Unify on Seismic native API

---

## 11. Currency Support Differences

### Seismic Compat Supports
| Type | Currencies |
|------|-----------|
| **Fiat** | BRL, COP, EUR, GBP, MXN, USD |
| **Stablecoin** | USDC |

### Seismic Compat Does NOT Support
| Currency | Workaround |
|----------|-----------|
| DAI | Use USDC |
| EURC | Use EUR (fiat) |
| PYUSD | Use USDC |
| USDB | Use USDC |
| USDT | Use USDC |

---

## 12. Transfer States (Bridge Compat)

```
awaiting_funds → funds_received → payment_submitted → payment_processed
     ↘              ↗
   in_review
     ↘
   undeliverable | returned → refund_in_flight → refunded
     ↘
   refund_failed

Or:
awaiting_funds → canceled
```

### Payment Rails Supported
| Type | Rails |
|------|-------|
| **Fiat** | ach, ach_push, ach_same_day, wire, swift, sepa, faster_payments, pix, spei, bre_b, co_bank_transfer |
| **Crypto** | base, ethereum, arbitrum, optimism, polygon, solana, avalanche_c_chain, celo, stellar, tron, tempo |

### Unsupported Rails
- `bridge_wallet` → returns `422 unsupported_rail`
