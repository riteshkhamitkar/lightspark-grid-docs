# Seismic Orchestration — API Reference: Customers & Accounts

> **Full Documentation Analysis** | **Source:** https://orchestration.seismic.systems/docs/api/customers, /docs/api/accounts  
> **Analyzed:** June 9, 2026

---

## PART A: CUSTOMERS API

### Base Path: `/customers/*`

These are **end-user customers** — individuals or businesses that YOUR product onboards onto the platform. They are scoped to your org. They are NOT the same as your org's own KYB onboarding (which lives under `/orgs`).

**Polymorphic surface**: A Customer carries a `type` discriminator (`"individual"` or `"business"`); profile data lives in a nested `profile` sub-object whose shape is picked by `type`:
- **`"individual"`** → carries a `KycProfile`
- **`"business"`** → carries a `KybProfile`

Every endpoint accepts either kind on the same URL and returns the right shape based on `type`.

---

### A.1 Customer Lifecycle Endpoints

#### `POST /customers` — Create a customer
```bash
POST /customers
Api-Key: seismic_...
Content-Type: application/json

{
  "type": "individual",
  "profile": {
    "identity": {
      "first_name": "Alice",
      "last_name": "Liddell",
      "date_of_birth": "1990-04-12"
    },
    "contact": {
      "email": "alice@example.com"
    },
    "addresses": {
      "residential": {
        "line1": "123 Main St",
        "city": "Brooklyn",
        "state": "NY",
        "postal_code": "11201",
        "country": "US"
      }
    }
  }
}
```

**Response:**
```json
{
  "id": "f01e...",
  "type": "individual",
  "profile": { ... },
  "kyc": {
    "status": "not_started",
    "link": { "token": "...", "url": "https://..." }
  },
  "created_at": "2026-01-15T10:30:00Z"
}
```

**Key behaviors:**
- Returns the canonical customer `id` (UUID)
- Auto-mints a `kyc_link` at create time
- `kyc.status` starts as `"not_started"`

#### `GET /customers/:id` — Read a customer
- Returns full customer envelope: `type`, `profile`, inline `kyc_link`
- `GET /customers/:id/kyc/profile` is an alias

#### `PUT /customers/:id/kyc/profile` — Upsert KYC profile
- Accepts **partial drafts**
- Returns the same envelope
- Can be called multiple times to build up the profile

#### `POST /customers/:id/kyc/submit` — Submit for verification
- Triggers KYC review (for individuals) or KYB review (for businesses)
- Status changes: `not_started` → `submitted`

#### `GET /customers/:id/kyc/state` — Read verification state
- Poll this to check KYC progress
- Status values: `not_started`, `submitted`, `under_review`, `approved`, `rejected`, `unknown`

#### `POST /customers/:id/kyc/resync` — Force fresh sync
- Force a fresh sync of the cached verification verdict

---

### A.2 Hosted KYC Link

Every customer carries a `kyc_link` inline on every read. Auto-minted at create-time; no separate endpoint needed to fetch it.

#### `POST /customers/:id/kyc-link/rotate` — Rotate KYC link token
- Overwrite the token in place after a suspected leak
- Returns new `{ token, url }`

---

### A.3 Per-Customer Account Endpoints

#### `POST /customers/:id/accounts/fiat/virtual` — Create fiat virtual account
#### `POST /customers/:id/accounts/fiat/external` — Register fiat external account
#### `POST /customers/:id/accounts/crypto/virtual` — Create crypto virtual account
#### `POST /customers/:id/accounts/crypto/external` — Register crypto external account

See PART B (Accounts) for detailed account creation.

---

### A.4 Customer Types

#### Customer (Individual)
```typescript
{
  id: string;                    // UUID
  type: "individual";
  profile: KycProfile;
  kyc: {
    status: KyStatus;
    link: HostedKycLink;
  };
  created_at: string;
  updated_at: string;
}
```

#### Customer (Business)
```typescript
{
  id: string;
  type: "business";
  profile: KybProfile;
  kyb: {
    status: KyStatus;
    link: HostedKycLink;
  };
  created_at: string;
  updated_at: string;
}
```

#### KycProfile (Individual)
```typescript
{
  identity: {
    first_name: string;
    last_name: string;
    date_of_birth: string;       // YYYY-MM-DD
    nationality?: string;         // ISO alpha-3
  };
  contact: {
    email?: string;
    phone?: string;               // E.164 format
  };
  addresses: {
    residential: Address;
  };
  employment?: {
    occupation?: string;          // SOC code or free text
    employment_status?: string;
  };
  risk?: {
    source_of_funds?: string;
    account_purpose?: string;
  };
  volumes?: {
    expected_monthly_volume?: string;
  };
  jurisdictions?: {
    tax_id?: string;
    tax_id_country?: string;      // ISO alpha-3
  };
  attestations?: {
    terms_accepted?: boolean;
    // ...
  };
  documents?: {
    // Document references
  };
}
```

#### KybProfile (Business)
```typescript
{
  identity: {
    legal_name: string;
    entity_type: string;
    tax_id?: string;
    formation_date?: string;
    registration_number?: string;
  };
  addresses: {
    registered: Address;
    physical?: Address;
  };
  contact: {
    email?: string;
    phone?: string;
    website?: string;
  };
  operations?: {
    jurisdictions?: string[];
    source_of_funds?: string;
    regulated_status?: string;
  };
  volumes?: {
    expected_monthly_volume?: string;
  };
  associated_persons?: AssociatedPerson[];
  documents?: {
    // Business documents
  };
}
```

#### HostedKycLink
```typescript
{
  token: string;       // Unique token for the hosted session
  url: string;         // Full URL to redirect the user to
}
```

#### Address
```typescript
{
  line1: string;
  line2?: string;
  city: string;
  state?: string;       // Province/region
  postal_code: string;
  country: string;      // ISO alpha-3 (e.g., "US", "IND", "GBR")
}
```

#### TransliteratedAddress
- Same fields as Address but for non-Latin script countries
- Used when the original address is in a different script

#### KyStatus
```typescript
"not_started" | "submitted" | "under_review" | "approved" | "rejected" | "unknown"
```

#### CustomerState
```typescript
{
  customer_id: string;
  status: KyStatus;
  // Additional state metadata
}
```

---

### A.5 Documents

Customers can have associated documents for KYC/KYB:
- Identity documents (passport, driver's license, national ID)
- Proof of address documents
- Business registration documents
- Upload via the hosted KYC link flow

---

## PART B: ACCOUNTS API

### Base Path: `/accounts/*` (org-wide) + `/customers/:id/accounts/*` (per-customer)

Accounts are **first-party**: every account attached under a customer is that customer's own funding/payout instrument. There is no beneficiary field because the customer IS the beneficiary.

**To send money to a third party:** Onboard them as a separate Customer and attach their account there. (Inline crypto destinations on a quote are the only exception.)

---

### B.1 Org-Wide Account Endpoints

#### `GET /accounts` — List all accounts
- Filterable by: customer, kind, origin, currency, rail, status
- Returns up to 200 rows

#### `GET /accounts/:id` — Read single account
- Polymorphic — branch on `kind` to dispatch on the layout of `details`

---

### B.2 Per-Customer Account Creation

#### `POST /customers/:id/accounts/fiat/virtual` — Provision virtual fiat account
```bash
POST /customers/:id/accounts/fiat/virtual
{
  "currency": "USD",
  "rails": ["ach", "wire"],
  "schema": "us_bank"
}
```
- Creates a Seismic-managed virtual bank account
- Returns account details with routing number, account number

#### `POST /customers/:id/accounts/fiat/external` — Register existing fiat account
```bash
POST /customers/:id/accounts/fiat/external
{
  "currency": "USD",
  "rails": ["ach"],
  "details": {
    "account_number": "...",
    "routing_number": "..."
  }
}
```

#### `POST /customers/:id/accounts/crypto/virtual` — Provision virtual crypto wallet
```bash
POST /customers/:id/accounts/crypto/virtual
{
  "currency": "USDC",
  "rails": ["base"],
  "schema": "evm"
}
```
- Creates a Seismic-managed crypto wallet
- Returns wallet address

#### `POST /customers/:id/accounts/crypto/external` — Register existing crypto wallet
```bash
POST /customers/:id/accounts/crypto/external
{
  "currency": "USDC",
  "rails": ["base"],
  "address": "0x..."
}
```

---

### B.3 Per-Customer Account Reading

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/customers/:id/accounts` | List accounts for customer |
| `GET` | `/customers/:id/accounts/fiat/:account_id` | Read fiat account |
| `GET` | `/customers/:id/accounts/crypto/:account_id` | Read crypto account |
| `GET` | `/customers/:id/accounts/requests/:request_id` | Poll account creation request |
| `GET` | `/customers/:id/accounts/requests?idempotency_key=...` | Look up by idempotency key |

---

### B.4 Account Shape

Every Account carries:
- `kind`: `"fiat"` or `"crypto"`
- `shape`: `"us"`, `"uk"`, `"iban"`, `"in_upi"`, `"evm"`, `"sol"`, ...
- `details`: Shape-specific details

#### FiatAccount
```typescript
{
  id: string;
  kind: "fiat";
  shape: "us" | "uk" | "iban" | "in_upi" | ...;
  currency: string;
  rails: string[];
  details: {
    account_number?: string;
    routing_number?: string;    // US only
    iban?: string;               // IBAN countries
    sort_code?: string;          // UK only
    // ... shape-specific
  };
  status: "active" | "pending" | "closed";
  created_at: string;
}
```

#### CryptoAccount
```typescript
{
  id: string;
  kind: "crypto";
  shape: "evm" | "sol" | ...;
  currency: string;              // USDC, USDT, etc.
  rails: string[];               // ["base"], ["ethereum"], etc.
  address: string;               // On-chain address
  status: "active" | "pending" | "closed";
  created_at: string;
}
```

#### FiatShape
| Shape | Description | Rails |
|-------|-------------|-------|
| `us` | US bank account | `ach`, `wire`, `ach_push`, `ach_same_day` |
| `uk` | UK bank account | `faster_payments` |
| `iban` | IBAN (EUR) | `sepa`, `swift` |
| `in_upi` | India UPI | `upi` |
| `br_pix` | Brazil PIX | `pix` |
| `mx_clabe` | Mexico CLABE | `spei` |
| `co_bank` | Colombia bank | `co_bank_transfer` |

#### CryptoShape
| Shape | Description | Chains |
|-------|-------------|--------|
| `evm` | Ethereum Virtual Machine | `base`, `ethereum`, `arbitrum`, `optimism`, `polygon`, `avalanche_c_chain` |
| `sol` | Solana | `solana` |
| `celo` | Celo | `celo` |
| `stellar` | Stellar | `stellar` |
| `tron` | Tron | `tron` |
| `tempo` | Tempo | `tempo` |

---

### B.5 Account Creation Request

For async account creation:
```typescript
{
  id: string;              // Request ID for polling
  status: "pending" | "completed" | "failed";
  account_id?: string;     // Set when completed
  error?: { code: string; message: string };
}
```

---

## C. End-User Onboarding Flow (3 Steps)

```
Step 1: CREATE customer
    │
    ▼
POST /customers
    │
    ├── Returns customer id + auto-minted kyc_link
    │
    ▼
Step 2: VERIFY customer
    │
    ├── Option A: Hosted KYC flow (recommended)
    │   Redirect user to kyc_link.url
    │   User completes verification in Seismic-hosted UI
    │   Poll GET /customers/:id/kyc/state until approved
    │
    ├── Option B: Inline verification
    │   PUT /customers/:id/kyc/profile (build profile)
    │   POST /customers/:id/kyc/submit
    │   Poll GET /customers/:id/kyc/state until approved
    │
    ▼
Step 3: CREATE account
    │
    ▼
POST /customers/:id/accounts/fiat/virtual   (for fiat)
POST /customers/:id/accounts/crypto/virtual (for crypto)
    │
    └── Account ready for quotes/transfers
```

---

## D. Comparison: Enso's Current KYC vs Seismic's KYC

| Aspect | Enso Current (Sumsub + Bridge) | Seismic |
|--------|-------------------------------|---------|
| ID Verification | Sumsub SDK (mobile) | Hosted KYC link (redirect) |
| Questionnaire | Custom form → Enso backend | PUT /customers/:id/kyc/profile |
| TOS | Bridge WebView | Built into hosted KYC or attestations |
| Submission | Auto-submit to Bridge | POST /customers/:id/kyc/submit |
| Status polling | Enso polls Sumsub + Bridge | Poll GET /customers/:id/kyc/state |
| Webhooks | Sumsub webhook + Bridge webhook | Seismic webhooks (customer.kyc.status_changed) |

### Key Differences for Migration

1. **No Sumsub SDK needed** — Seismic provides hosted KYC link
2. **No separate questionnaire system** — Profile is built via API
3. **No TOS WebView** — Terms acceptance part of KYC profile or hosted flow
4. **Single webhook** — `customer.kyc.status_changed` replaces Sumsub + Bridge webhooks
5. **No KycVerification table needed** — Status tracked by Seismic

### Migration Strategy for Existing Users

**Users with ACTIVE BridgeCustomer:**
- These users are already KYC-approved with Bridge
- Seismic's Bridge compat layer may recognize existing Bridge customers
- Need to investigate if Seismic can import/recognize existing Bridge KYC data

**Users in-progress (Sumsub but not yet Bridge):**
- Need to re-route through Seismic's KYC flow
- Sumsub applicant data may need to be transferred

**Data mapping:**
| Enso KycProfile | Seismic KycProfile |
|-----------------|-------------------|
| `firstName` | `identity.first_name` |
| `lastName` | `identity.last_name` |
| `dateOfBirth` | `identity.date_of_birth` |
| `nationality` | `identity.nationality` |
| `phone` | `contact.phone` |
| `streetLine1` | `addresses.residential.line1` |
| `city` | `addresses.residential.city` |
| `state` | `addresses.residential.state` |
| `postalCode` | `addresses.residential.postal_code` |
| `country` | `addresses.residential.country` |
| `taxId` | `jurisdictions.tax_id` |
| `taxIdCountry` | `jurisdictions.tax_id_country` |
| `occupation` | `employment.occupation` |
| `employmentStatus` | `employment.employment_status` |
| `sourceOfFunds` | `risk.source_of_funds` |
| `accountPurpose` | `risk.account_purpose` |
| `expectedMonthlyVolume` | `volumes.expected_monthly_volume` |
| `actingAsIntermediary` | (part of attestations) |
