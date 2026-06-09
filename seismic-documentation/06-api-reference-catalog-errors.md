# Seismic Orchestration — API Reference: Catalog & Errors

> **Full Documentation Analysis** | **Source:** https://orchestration.seismic.systems/docs/api/rates, /rails, /schemas, /currencies, /errors  
> **Analyzed:** June 9, 2026

---

## PART A: CATALOG ENDPOINTS (Unauthenticated Reference)

These endpoints provide metadata about supported rails, currencies, and schemas. They require **no authentication**.

---

### A.1 `GET /rates` — Mid-Market Reference Rates

Returns mid-market reference rates. No quote required.

```bash
GET /rates?from=USD&to=EUR

Response: {
  "from": "USD",
  "to": "EUR",
  "rate": "0.92",
  "timestamp": "2026-01-15T10:30:00Z"
}
```

**Use case:** Display indicative rates in your UI before the user requests a formal quote.

---

### A.2 `GET /rails` — Supported Payment Rails

Returns the full list of supported payment rails.

```bash
GET /rails

Response: [
  { "id": "ach", "kind": "fiat", "name": "ACH" },
  { "id": "wire", "kind": "fiat", "name": "Wire Transfer" },
  { "id": "sepa", "kind": "fiat", "name": "SEPA" },
  { "id": "pix", "kind": "fiat", "name": "PIX" },
  { "id": "base", "kind": "crypto", "name": "Base" },
  { "id": "ethereum", "kind": "crypto", "name": "Ethereum" },
  { "id": "solana", "kind": "crypto", "name": "Solana" },
  // ...
]
```

### Fiat Rails
| Rail | Description | Countries |
|------|-------------|-----------|
| `ach` | Automated Clearing House | US |
| `ach_push` | ACH Push | US |
| `ach_same_day` | Same-day ACH | US |
| `wire` | Wire Transfer | US |
| `swift` | SWIFT | International |
| `sepa` | SEPA Transfer | EU |
| `faster_payments` | Faster Payments | UK |
| `pix` | PIX | Brazil |
| `spei` | SPEI | Mexico |
| `upi` | UPI | India |
| `bre_b` | BRE B | TBD |
| `co_bank_transfer` | Colombia Bank Transfer | Colombia |

### Crypto Rails (Supported)
| Rail | Chain Type |
|------|-----------|
| `base` | EVM (L2) |
| `ethereum` | EVM (L1) |
| `arbitrum` | EVM (L2) |
| `optimism` | EVM (L2) |
| `polygon` | EVM (L2) |
| `solana` | Solana |
| `avalanche_c_chain` | EVM (L1) |
| `celo` | EVM (L1) |
| `stellar` | Stellar |
| `tron` | Tron |
| `tempo` | Tempo |

---

### A.3 `GET /schemas` — Rail Groups (Schemas)

Named groups of rails that can be used in quote requests.

```bash
GET /schemas

Response: [
  { "id": "us_bank", "rails": ["ach", "wire", "swift"] },
  { "id": "evm", "rails": ["base", "ethereum", "polygon", "arbitrum"] },
  { "id": "all_fiat", "rails": ["ach", "wire", "sepa", "pix", ...] }
]
```

### Common Schemas
| Schema | Expands To | Use Case |
|--------|-----------|----------|
| `us_bank` | `["ach", "wire", "swift"]` | All US bank rails |
| `evm` | `["base", "ethereum", "polygon", "arbitrum"]` | All EVM chains |
| `all_fiat` | All fiat rails | Fan out across all fiat |
| `all_crypto` | All crypto rails | Fan out across all crypto |

**Usage in quotes:**
```typescript
destination: {
  currency: "USDC",
  schemas: ["evm"]  // Price on all EVM chains
}
```

---

### A.4 `GET /currencies` — Supported Currencies

```bash
GET /currencies

Response: [
  { "code": "USD", "name": "US Dollar", "kind": "fiat" },
  { "code": "EUR", "name": "Euro", "kind": "fiat" },
  { "code": "USDC", "name": "USD Coin", "kind": "stablecoin" },
  // ...
]
```

### Supported Fiat Currencies
| Currency | Name |
|----------|------|
| `USD` | US Dollar |
| `EUR` | Euro |
| `GBP` | British Pound |
| `BRL` | Brazilian Real |
| `COP` | Colombian Peso |
| `MXN` | Mexican Peso |

### Supported Stablecoins
| Currency | Name | Chains |
|----------|------|--------|
| `USDC` | USD Coin | Base, Ethereum, Polygon, Solana, etc. |

### Not Supported (Bridge lists these; Seismic does NOT)
- `DAI`, `EURC`, `PYUSD`, `USDB`, `USDT`

---

## PART B: ERRORS

### B.1 Error Envelope

Every non-2xx response uses the same JSON envelope:

```json
{
  "ok": false,
  "error": {
    "code": "AccountNotFound",
    "message": "No account found for id acc_1a2b"
  }
}
```

- **`ok`** — literal `false`, present only on errors. Success responses are flat payloads with no `ok` field.
- **`code`** — stable PascalCase enum value. Safe to switch on programmatically.
- **`message`** — human-readable detail. Safe to surface to end users.

### B.2 Status Codes

| Status | Meaning | Typical Causes |
|--------|---------|---------------|
| `400` | Bad request | Malformed body, invalid enum, rule violation |
| `401` | Unauthenticated | Missing/expired access token — refresh and retry |
| `403` | Not allowed | Authed, but you don't own the resource |
| `404` | Not found | Unknown id, or scoped-away |
| `409` | Conflict | Duplicate row or idempotency violation |
| `422` | Unprocessable entity | Semantic failure — no eligible options, snapshot not ready |
| `5xx` | Server-side | Upstream outage, DB issue. Safe to retry. |

### B.3 Error Codes by Surface

#### Orgs + KYB
| Code | Status | Meaning |
|------|--------|---------|
| `OrgNotFound` | 404 | Unknown org id (or credential isn't scoped to it) |
| `NotOrgMember` | 403 | Authed, but user isn't a member of requested org |
| `LastOwner` | 409 | Demote/remove would leave org with zero owners |
| `KybProfileIncomplete` | 422 | Required KYB fields missing — see `missing` array |
| `KybNotApproved` | 422 | Action requires org to be KYB-approved |

#### Auth
| Code | Status | Meaning |
|------|--------|---------|
| `PasswordTooShort` | 400 | Password below minimum length |
| `EmailAlreadyExists` | 409 | Signup email already registered |
| `InvalidCredentials` | 401 | Bad email/password on login |
| `MfaChallengeInvalid` | 401 | Wrong code, expired challenge, clock skew > 30s |
| `RefreshTokenInvalid` | 401 | Refresh token expired or revoked |

#### API Keys
| Code | Status | Meaning |
|------|--------|---------|
| `BothCredsProvided` | 400 | Both `Api-Key:` and `Authorization: Bearer` headers sent |
| `ApiKeyNotFound` | 404 | Unknown key id (or belongs to different org) |
| `ApiKeyRevoked` | 401 | Key was valid but is now revoked |
| `AlreadyRevoked` | 409 | Revoke called on already-revoked key |

#### Customers + Accounts
| Code | Status | Meaning |
|------|--------|---------|
| `CustomerNotFound` | 404 | Unknown id (or belongs to different org) |
| `EndUserCustomerExists` | 409 | This (org, reference_id) pair already onboarded |
| `KycNotApproved` | 422 | Account creation requires `kyc.status === "approved"` |
| `AccountNotFound` | 404 | Unknown account id |
| `RailNotAllowed` | 400 | Rail not settleable for chosen currency |
| `MissingAccountFields` | 400 | Required field absent (e.g., account_number) |
| `InvalidAccountFields` | 400 | Field present but invalid (format failure) |

#### Quotes
| Code | Status | Meaning |
|------|--------|---------|
| `NoEligibleOptions` | 422 | No option priced this corridor |
| `QuoteNotFound` | 404 | Snapshot id unknown (or not yours) |
| `QuoteNotReady` | 422 | Snapshot still pending — poll GET |
| `QuoteExpired` | 422 | Quote's expires_at has passed |
| `QuoteIdInvalid` | 400 | quote_id not in snapshot.quotes[] |
| `AlreadyAccepted` | 409 | Snapshot already turned into order |
| `AmbiguousRail` | 400 | Account supports multiple rails; none specified |
| `SchemaNotApplicable` | 400 | Schema kind doesn't match leg's account family |
| `SchemaNotFound` | 400 | Unknown schema id |

#### Generic
| Code | Status | Meaning |
|------|--------|---------|
| `InvalidRequest` | 400 | Catch-all for malformed body/invalid enum |
| `Unauthorized` | 401 | Missing or invalid bearer token |
| `Forbidden` | 403 | Authed, but not permitted |
| `MfaRequired` | 403 | Resource needs MFA-active session |
| `InternalError` | 500 | Unexpected server failure. Retryable. |
| `UpstreamError` | 502 | Seismic dependency returned error. Retryable. |
| `Unknown` | — | Client-side fallback when response couldn't be parsed |

---

### B.4 Retry Guidance

| Status Range | Retry? | Strategy |
|-------------|--------|----------|
| `4xx` | **NO** | Fix the input. Idempotency shields from double-charges. |
| `5xx` | **YES** | Exponential backoff, capped at ~5 attempts |
| `401` | **YES** | Refresh access token and retry. If 401s again → send user to login |

### Special Cases
- **`QuoteExpired`** → Re-create snapshot, show new price, let user re-confirm
- **`AlreadyAccepted`** → Treat as success (prior request won the race)
- **`QuoteIdInvalid`** → UI sent invalid id. Log + report.

---

## C. Enso Integration Notes

### Catalog Usage

```typescript
// Pre-load catalog data for your UI
const [rails, currencies, schemas] = await Promise.all([
  api.rails.list(),
  api.currencies.list(),
  api.schemas.list()
]);

// Show indicative rate
const rate = await api.rates.get({ from: "USD", to: "USDC" });
```

### Error Handling Strategy

```typescript
try {
  const order = await api.quotes.acceptBest(snapshotId);
} catch (error) {
  if (error.code === "QuoteExpired") {
    // Re-create snapshot
    const newSnapshot = await api.quotes.create(request, { wait: true });
    // Show user updated pricing
  } else if (error.code === "AlreadyAccepted") {
    // Treat as success
    const order = await api.customers.order(orderId);
  } else if (error.status >= 500) {
    // Retry with backoff
    await retryWithBackoff(() => api.quotes.acceptBest(snapshotId));
  } else {
    // Log and surface to user
    reportError(error);
  }
}
```

### Important: Currency Support

Seismic supports **fewer currencies** than Bridge:
- **Fiat:** USD, EUR, GBP, BRL, COP, MXN
- **Stablecoins:** USDC only
- **NOT supported:** DAI, EURC, PYUSD, USDB, USDT

If Enso currently supports any of these unsupported currencies, plan to either:
1. Migrate users to supported alternatives
2. Maintain dual integration (Bridge for unsupported, Seismic for supported)
3. Request Seismic to add support
