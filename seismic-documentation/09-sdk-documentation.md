# Seismic Orchestration — SDK Documentation

> **Full Documentation Analysis** | **Source:** https://orchestration.seismic.systems/docs/sdk/overview  
> **Analyzed:** June 9, 2026

---

## 1. SDK Overview

Every SDK wraps the REST surface. Shared class name (`SeismicOrchestrationClient`), method shapes, and error codes across languages.

### Availability
| Language | Package | Status | Docs |
|----------|---------|--------|------|
| **TypeScript** | `seismic-orchestration` (npm) | ✅ Available | Full reference |
| Python | — | ⏳ Request | Open issue |
| Go | — | ⏳ Request | Open issue |
| Rust | — | ⏳ Request | Open issue |
| Ruby | — | ⏳ Request | Open issue |
| Java | — | ⏳ Request | Open issue |

For additional languages: https://github.com/SeismicSystems/ramps/issues

---

## 2. Installation

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

**Dependencies:**
- Single runtime dependency: `zod`
- No heavy framework dependencies

---

## 3. What the SDK Provides

### 3.1 Single-Flight Refresh-on-401
If 50 concurrent requests 401 at the same time, the client refreshes **once**, not 50 times.

```
Request 1  ──┐
Request 2  ──┼──► 401 detected ──► ONE refresh call ──► Retry all 50
Request 3  ──┤                              ▲
...           │                              │
Request 50 ──┘                              │
                                            │
                    All share one refresh ◄─┘
```

### 3.2 Runtime Response Validation
All responses validated against Zod schemas mirrored from server DTOs:
```typescript
const customer = await api.customers.create({ ... });
// customer is fully typed — TypeScript knows all fields
// Runtime validation ensures the shape matches expectations
```

### 3.3 Normalized Request Shapes
Clean, typed interfaces for common operations:
```typescript
// Instead of raw HTTP:
fetch("/api/v0/customers", { method: "POST", body: JSON.stringify({ ... }) })

// Use typed SDK:
api.customers.create({
  type: "individual",
  profile: { ... }
})
```

---

## 4. SeismicOrchestrationClient

### Constructor Options
```typescript
interface ClientOptions {
  // Auth (pick ONE)
  apiKey?: string;           // Server-to-server (recommended)
  tokenStore?: TokenStore;   // Dashboard/browser sessions
  
  // Environment
  baseURL?: string;          // Override default base URL
}
```

### Factory Methods
```typescript
// Sandbox environment
const api = SeismicOrchestrationClient.sandbox({
  apiKey: "seismic_sandbox_..."
});

// Production environment
const api = SeismicOrchestrationClient.production({
  apiKey: "seismic_prod_..."
});
```

### API Namespaces
```typescript
api.orgs          // Org management, KYB
api.auth          // Signup, login, MFA, API keys
api.customers     // Customer CRUD, KYC, accounts
api.accounts      // Account listing, reading
api.quotes        // Quote creation, acceptance
api.rates         // Reference rates
api.rails         // Supported rails
api.schemas       // Rail schemas
api.currencies    // Supported currencies
```

---

## 5. SDK Usage Examples

### Create Customer
```typescript
const customer = await api.customers.create({
  type: "individual",
  profile: {
    identity: {
      first_name: "Alice",
      last_name: "Liddell",
      date_of_birth: "1990-04-12"
    },
    contact: {
      email: "alice@example.com"
    },
    addresses: {
      residential: {
        line1: "123 Main St",
        city: "Brooklyn",
        state: "NY",
        postal_code: "11201",
        country: "US"
      }
    }
  }
});

console.log(customer.id);        // UUID
console.log(customer.kyc.link);  // Hosted KYC URL
```

### Update KYC Profile
```typescript
await api.customers.updateKycProfile(customer.id, {
  employment: {
    occupation: "Software Developer",
    employment_status: "employed"
  },
  risk: {
    source_of_funds: "salary",
    account_purpose: "personal_use"
  }
});
```

### Submit KYC
```typescript
await api.customers.submitKyc(customer.id);
```

### Poll KYC Status
```typescript
const state = await api.customers.kycState(customer.id);
console.log(state.status); // "not_started" | "submitted" | "under_review" | "approved" | "rejected"
```

### Create Account
```typescript
// Fiat virtual account (ACH + Wire)
const account = await api.customers.createFiatVirtualAccount(customer.id, {
  currency: "USD",
  rails: ["ach", "wire"],
  schema: "us_bank"
});

// Crypto virtual account (Base chain)
const cryptoAccount = await api.customers.createCryptoVirtualAccount(customer.id, {
  currency: "USDC",
  rails: ["base"],
  schema: "evm"
});
```

### Request Quotes
```typescript
// Fan out across all USD rails
const snapshot = await api.quotes.create({
  end_customer_id: customer.id,
  end_customer_type: "individual",
  amount: "1000",
  source: { currency: "USD" },  // Omit rails to fan out
  destination: {
    currency: "USDC",
    rails: ["base"]
  }
}, { wait: true });

// Display options
for (const quote of snapshot.quotes) {
  console.log(`${quote.in.rail}: ${quote.out.amount} ${quote.out.currency}`);
}

// Accept best
const order = await api.quotes.acceptBest(snapshot.id);
```

### Read Order
```typescript
const order = await api.customers.order(orderId);
console.log(order.status);
```

### List Orders
```typescript
const recentOrders = await api.customers.orders();
// Returns up to 200 rows, most-recent first
```

---

## 6. Enso Backend Integration

### Initialization
```typescript
// src/modules/seismic/seismic.provider.ts
import { SeismicOrchestrationClient } from "seismic-orchestration";

export class SeismicProvider {
  private api: ReturnType<typeof SeismicOrchestrationClient.sandbox>;
  
  constructor() {
    this.api = SeismicOrchestrationClient.sandbox({
      apiKey: process.env.SEISMIC_API_KEY!
    });
  }
  
  // Customer management
  async createCustomer(profile: KycProfile) {
    return this.api.customers.create({
      type: "individual",
      profile: {
        identity: {
          first_name: profile.firstName,
          last_name: profile.lastName,
          date_of_birth: profile.dateOfBirth,
          nationality: profile.nationality
        },
        contact: {
          email: profile.email,
          phone: profile.phone
        },
        addresses: {
          residential: {
            line1: profile.streetLine1,
            line2: profile.streetLine2,
            city: profile.city,
            state: profile.state,
            postal_code: profile.postalCode,
            country: profile.country
          }
        }
      }
    });
  }
  
  // Quotes
  async getQuotes(request: QuoteRequest) {
    return this.api.quotes.create(request, { wait: true });
  }
  
  async acceptBestQuote(snapshotId: string) {
    return this.api.quotes.acceptBest(snapshotId);
  }
}
```

### Environment Setup
```bash
# .env
SEISMIC_API_KEY=seismic_sandbox_abc123...
SEISMIC_BASE_URL=https://orchestration-sandbox.seismictest.net/api
```

### Module Registration
```typescript
// src/app.module.ts
import { SeismicModule } from './modules/seismic/seismic.module';

@Module({
  imports: [
    // ... existing modules
    SeismicModule,
  ],
})
export class AppModule {}
```

---

## 7. SDK Reference: Complete Method List

### Orgs
| Method | Endpoint | Description |
|--------|----------|-------------|
| `api.orgs.list()` | `GET /orgs` | List orgs |
| `api.orgs.create()` | `POST /orgs` | Create org |
| `api.orgs.retrieve()` | `GET /org` | Read current org |
| `api.orgs.update()` | `PATCH /org` | Update org |
| `api.orgs.members.list()` | `GET /org/members` | List members |
| `api.orgs.members.invite()` | `POST /org/members` | Invite member |
| `api.orgs.members.update()` | `PATCH /org/members/{id}` | Update role |
| `api.orgs.members.remove()` | `DELETE /org/members/{id}` | Remove member |
| `api.orgs.kyb.read()` | `GET /org/kyb` | Read KYB state |
| `api.orgs.kyb.updateProfile()` | `PUT /org/kyb/profile` | Update KYB profile |
| `api.orgs.kyb.submit()` | `POST /org/kyb/submit` | Submit KYB |
| `api.orgs.kyb.state()` | `GET /org/kyb/state` | Read KYB status |

### Auth
| Method | Endpoint | Description |
|--------|----------|-------------|
| `api.auth.signup()` | `POST /auth/signup` | Create account |
| `api.auth.login()` | `POST /auth/login` | Login |
| `api.auth.mfa.verify()` | `POST /auth/mfa/verify` | Verify MFA |
| `api.auth.mfa.enroll()` | `POST /auth/mfa/enroll` | Enroll MFA |
| `api.auth.mfa.activate()` | `POST /auth/mfa/activate` | Activate MFA |
| `api.auth.refresh()` | `POST /auth/refresh` | Refresh token |
| `api.auth.logout()` | `POST /auth/logout` | Logout |
| `api.auth.me()` | `GET /auth/me` | Whoami |
| `api.auth.apiKeys.mint()` | `POST /auth/api-keys` | Mint API key |
| `api.auth.apiKeys.list()` | `GET /auth/api-keys` | List keys |
| `api.auth.apiKeys.revoke()` | `DELETE /auth/api-keys/{id}` | Revoke key |

### Customers
| Method | Endpoint | Description |
|--------|----------|-------------|
| `api.customers.create()` | `POST /customers` | Create customer |
| `api.customers.list()` | `GET /customers` | List customers |
| `api.customers.retrieve()` | `GET /customers/{id}` | Read customer |
| `api.customers.updateKycProfile()` | `PUT /customers/{id}/kyc/profile` | Update KYC |
| `api.customers.submitKyc()` | `POST /customers/{id}/kyc/submit` | Submit KYC |
| `api.customers.kycState()` | `GET /customers/{id}/kyc/state` | KYC status |
| `api.customers.resyncKyc()` | `POST /customers/{id}/kyc/resync` | Force resync |
| `api.customers.rotateKycLink()` | `POST /customers/{id}/kyc-link/rotate` | Rotate link |

### Accounts (per-customer)
| Method | Endpoint | Description |
|--------|----------|-------------|
| `api.customers.createFiatVirtualAccount()` | `POST .../fiat/virtual` | Create fiat virtual |
| `api.customers.createFiatExternalAccount()` | `POST .../fiat/external` | Register fiat external |
| `api.customers.createCryptoVirtualAccount()` | `POST .../crypto/virtual` | Create crypto virtual |
| `api.customers.createCryptoExternalAccount()` | `POST .../crypto/external` | Register crypto external |
| `api.customers.listAccounts()` | `GET /customers/{id}/accounts` | List accounts |
| `api.customers.order()` | `GET /customers/{id}/orders/{orderId}` | Read order |
| `api.customers.orders()` | `GET /customers/{id}/orders` | List orders |

### Quotes
| Method | Endpoint | Description |
|--------|----------|-------------|
| `api.quotes.create()` | `POST /quotes` | Create quote |
| `api.quotes.retrieve()` | `GET /quotes/{id}` | Read snapshot |
| `api.quotes.accept()` | `POST /quotes/{id}/accept` | Accept specific |
| `api.quotes.acceptBest()` | `POST /quotes/{id}/accept-best` | Accept best |

### Catalog
| Method | Endpoint | Description |
|--------|----------|-------------|
| `api.rates.get()` | `GET /rates` | Reference rate |
| `api.rails.list()` | `GET /rails` | Supported rails |
| `api.schemas.list()` | `GET /schemas` | Rail schemas |
| `api.currencies.list()` | `GET /currencies` | Currencies |

---

## 8. Error Handling in SDK

```typescript
import { SeismicError } from "seismic-orchestration";

try {
  const order = await api.quotes.acceptBest(snapshotId);
} catch (error) {
  if (error instanceof SeismicError) {
    console.log(error.code);      // "QuoteExpired"
    console.log(error.message);   // Human-readable
    console.log(error.status);    // HTTP status code
    
    switch (error.code) {
      case "QuoteExpired":
        // Re-create snapshot
        break;
      case "AlreadyAccepted":
        // Treat as success
        break;
      default:
        // Handle other errors
    }
  }
}
```
