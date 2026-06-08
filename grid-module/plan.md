# Lightspark Grid Payouts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a parallel `grid` NestJS module that replaces Bridge for fiat payouts while preserving Turnkey USDC + Relay signing UX.

**Architecture:** `GridProvider` (Basic auth HTTP client) + handlers for KYC, external accounts, and payouts. `KycProfile` feeds `POST /customers`; Grid kyc-link is a new partner step. Payouts use JIT quotes + Relay crypto funding + `POST /quotes/{id}/execute`. Webhooks drive terminal status.

**Tech Stack:** NestJS, Prisma/PostgreSQL, Axios, Relay Protocol, Lightspark Grid API (`2025-10-13`), Jest e2e

**Spec:** `docs/superpowers/specs/2026-06-08-lightspark-grid-payouts-design.md`

---

## File Map

| File | Responsibility |
|------|----------------|
| `prisma/schema.prisma` | Add Grid models + User/KycProfile relations |
| `src/config/validation.schema.ts` | `GRID_*` env validation |
| `.env.example` | Document Grid vars |
| `src/modules/grid/grid.types.ts` | Grid API TypeScript interfaces |
| `src/modules/grid/providers/grid.provider.ts` | HTTP client |
| `src/modules/grid/utils/grid-country.util.ts` | ISO alpha-3 ↔ alpha-2 |
| `src/modules/grid/utils/grid-account-type.mapper.ts` | Bridge accountType → Grid |
| `src/modules/grid/utils/grid-kyc-payload.mapper.ts` | KycProfile → Grid customer |
| `src/modules/grid/utils/grid-funding.util.ts` | Parse JIT paymentInstructions |
| `src/modules/grid/utils/grid-webhook-verify.util.ts` | HMAC verification |
| `src/modules/grid/utils/grid-error.util.ts` | Map Grid errors → HTTP |
| `src/modules/grid/guards/grid-webhook.guard.ts` | Webhook guard |
| `src/modules/grid/handlers/grid-kyc.handler.ts` | Customer create + kyc-link |
| `src/modules/grid/handlers/grid-external-account.handler.ts` | Payee CRUD |
| `src/modules/grid/handlers/grid-payout.handler.ts` | prepare/execute/status |
| `src/modules/grid/handlers/grid-webhook.handler.ts` | Webhook event routing |
| `src/modules/grid/webhooks/grid-webhook.types.ts` | Event payload types |
| `src/modules/grid/webhooks/grid-webhook.controller.ts` | `POST /grid/webhook` |
| `src/modules/grid/dto/*.ts` | Request DTOs |
| `src/modules/grid/grid.service.ts` | Facade |
| `src/modules/grid/grid.controller.ts` | Authenticated routes |
| `src/modules/grid/grid.module.ts` | Module wiring |
| `src/modules/kyc/config/partner-steps.config.ts` | Add `grid` partner |
| `src/modules/kyc/handlers/partner-submission.handler.ts` | `checkAndSubmitGrid` |
| `src/modules/kyc/kyc.module.ts` | Import GridModule (forwardRef) |
| `src/app.module.ts` | Register GridModule |
| `src/modules/balances/handlers/transaction-history.handler.ts` | Grid payout rows |

---

## Task 0: Sandbox Spike (manual — blocks funding parser)

**Files:** None (curl + notes in `docs/superpowers/specs/grid-spike-notes.md`)

- [ ] **Step 1: Set sandbox credentials**

```bash
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
export GRID_CLIENT_ID="<sandbox>"
export GRID_CLIENT_SECRET="<sandbox>"
```

- [ ] **Step 2: Create customer + fund + external account + JIT quote**

Run quickstart from `docs/lightspark-grid-docs/07-payouts-and-b2b/02-quickstart.md`, then:

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": { "sourceType": "JIT", "customerId": "Customer:..." },
    "destination": { "destinationType": "ACCOUNT", "accountId": "ExternalAccount:..." },
    "amount": "100.00",
    "amountCurrency": "MXN",
    "amountLock": "RECEIVING"
  }' | jq .
```

- [ ] **Step 3: Record `paymentInstructions` field names**

Save full JSON to `docs/superpowers/specs/grid-spike-notes.md`. Update `grid-funding.util.ts` parser in Task 8 to match **actual** fields.

- [ ] **Step 4: Send test webhook**

```bash
curl -X POST "$GRID_BASE_URL/sandbox/webhooks/send-test" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"eventType": "transaction.status_change"}'
```

Expected: event received at ngrok endpoint; note exact `X-Grid-Signature` format.

**Do not proceed to Task 8 funding parser until Step 3 is complete.**

---

### Task 1: Prisma schema — Grid models

**Files:**
- Modify: `prisma/schema.prisma`
- Create: `prisma/migrations/<timestamp>_add_grid_integration/migration.sql` (via `npx prisma migrate dev`)

- [ ] **Step 1: Add models to schema.prisma**

Append after `BridgeActivityEvent` block (copy from spec §7.1). Add to `User`:

```prisma
  GridCustomer        GridCustomer?
  GridActivityEvent   GridActivityEvent[]
```

Add to `KycProfile`:

```prisma
  gridCustomer        GridCustomer?
```

- [ ] **Step 2: Generate migration**

```bash
cd c:\Users\ADMIN\Desktop\anzo-backend
npx prisma migrate dev --name add_grid_integration
```

Expected: migration SQL creates 4 tables + indexes.

- [ ] **Step 3: Generate client**

```bash
npx prisma generate
```

Expected: `@prisma/client` includes `GridCustomer`, `GridInternalAccount`, `GridExternalAccount`, `GridTransaction`.

---

### Task 2: Environment & config

**Files:**
- Modify: `src/config/validation.schema.ts`
- Modify: `.env.example`
- Modify: `src/config/configuration.ts` (if explicit map exists — add grid section)

- [ ] **Step 1: Add Joi validation**

In `validation.schema.ts`, after Bridge block:

```typescript
GRID_ENABLED: Joi.string().valid('true', 'false').optional().default('false'),
GRID_BASE_URL: Joi.string().uri().optional()
  .default('https://api.lightspark.com/grid/2025-10-13'),
GRID_CLIENT_ID: Joi.string().optional(),
GRID_CLIENT_SECRET: Joi.string().optional(),
GRID_WEBHOOK_SECRET: Joi.string().optional(),
GRID_TRANSFER_DEVELOPER_FEE: Joi.string().optional().default('0.00'),
```

- [ ] **Step 2: Update .env.example**

```bash
# Lightspark Grid (fiat payouts)
# GRID_ENABLED=true
# GRID_BASE_URL=https://api.lightspark.com/grid/2025-10-13
# GRID_CLIENT_ID=
# GRID_CLIENT_SECRET=
# GRID_WEBHOOK_SECRET=
# GRID_TRANSFER_DEVELOPER_FEE=0.00
```

- [ ] **Step 3: Verify boot**

```bash
npm run build
```

Expected: compiles without env errors.

---

### Task 3: Grid types

**Files:**
- Create: `src/modules/grid/grid.types.ts`

- [ ] **Step 1: Create types file**

```typescript
export type GridCustomerType = 'INDIVIDUAL' | 'BUSINESS';

export interface GridAddress {
  line1: string;
  line2?: string;
  city: string;
  state?: string;
  postalCode: string;
  country: string;
}

export interface GridIdentification {
  type: 'SSN' | 'PASSPORT' | 'ITIN' | string;
  number: string;
}

export interface GridCreateCustomerPayload {
  customerType: GridCustomerType;
  platformCustomerId: string;
  region: string;
  currencies?: string[];
  fullName: string;
  birthDate: string;
  nationality: string;
  email?: string;
  phone?: string;
  address?: GridAddress;
  identification?: GridIdentification;
}

export interface GridCustomer {
  id: string;
  platformCustomerId?: string;
  customerType: GridCustomerType;
  region: string;
  kycStatus: string;
  fullName?: string;
  email?: string;
  createdAt: string;
  updatedAt: string;
}

export interface GridKycLink {
  url: string;
  token: string;
  expiresAt: string;
}

export interface GridInternalAccount {
  id: string;
  type: string;
  currency: string;
  balance?: string;
  status: string;
  paymentInstructions?: Record<string, unknown>;
}

export interface GridExternalAccountPayload {
  accountType: string;
  currency: string;
  accountOwnerName: string;
  accountNumber?: string;
  routingNumber?: string;
  iban?: string;
  clabe?: string;
  pixKey?: string;
  pixKeyType?: string;
  upiId?: string;
  sortCode?: string;
  [key: string]: unknown;
}

export interface GridExternalAccount {
  id: string;
  accountType: string;
  currency: string;
  accountOwnerName: string;
  status: string;
}

export type GridQuoteSourceType = 'ACCOUNT' | 'JIT';
export type GridAmountLock = 'SENDING' | 'RECEIVING';

export interface GridCreateQuotePayload {
  source: {
    sourceType: GridQuoteSourceType;
    accountId?: string;
    customerId?: string;
  };
  destination: {
    destinationType: 'ACCOUNT' | 'UMA_ADDRESS';
    accountId?: string;
    umaAddress?: string;
  };
  amount: string;
  amountCurrency: string;
  amountLock?: GridAmountLock;
}

export interface GridQuote {
  id: string;
  sourceAmount: string;
  sourceAmountCurrency: string;
  destinationAmount: string;
  destinationAmountCurrency: string;
  exchangeRate?: string;
  fees?: string;
  feesCurrency?: string;
  totalCost?: string;
  expiresAt: string;
  status: string;
  paymentInstructions?: Record<string, unknown>;
}

export interface GridTransaction {
  id: string;
  status: string;
  quoteId?: string;
  amount?: string;
  currency?: string;
}

export interface GridWebhookEnvelope<T = Record<string, unknown>> {
  eventType: string;
  timestamp: string;
  data: T;
}
```

- [ ] **Step 2: Build**

```bash
npm run build
```

Expected: PASS (file not imported yet — no errors).

---

### Task 4: GridProvider

**Files:**
- Create: `src/modules/grid/providers/grid.provider.ts`
- Create: `src/modules/grid/providers/grid.provider.spec.ts`
- Test: `src/modules/grid/providers/grid.provider.spec.ts`

- [ ] **Step 1: Write failing provider test**

```typescript
import { Test } from '@nestjs/testing';
import { ConfigService } from '@nestjs/config';
import { GridProvider } from './grid.provider';

describe('GridProvider', () => {
  it('isEnabled returns false when GRID_ENABLED is false', () => {
    const provider = new GridProvider({
      get: (key: string, def?: string) =>
        key === 'GRID_ENABLED' ? 'false' : def,
    } as ConfigService);
    expect(provider.isEnabled()).toBe(false);
  });

  it('isEnabled returns true when GRID_ENABLED is true and credentials set', () => {
    const provider = new GridProvider({
      get: (key: string, def?: string) => {
        const map: Record<string, string> = {
          GRID_ENABLED: 'true',
          GRID_CLIENT_ID: 'id',
          GRID_CLIENT_SECRET: 'secret',
          GRID_BASE_URL: 'https://api.lightspark.com/grid/2025-10-13',
        };
        return map[key] ?? def;
      },
    } as ConfigService);
    expect(provider.isEnabled()).toBe(true);
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npx jest src/modules/grid/providers/grid.provider.spec.ts --no-cache
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement GridProvider**

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios, { AxiosInstance } from 'axios';
import {
  GridCreateCustomerPayload,
  GridCreateQuotePayload,
  GridCustomer,
  GridExternalAccount,
  GridExternalAccountPayload,
  GridInternalAccount,
  GridKycLink,
  GridQuote,
  GridTransaction,
} from '../grid.types';

@Injectable()
export class GridProvider {
  private readonly logger = new Logger(GridProvider.name);
  private readonly client: AxiosInstance;
  private readonly enabled: boolean;

  constructor(private readonly configService: ConfigService) {
    const baseURL = this.configService.get<string>(
      'GRID_BASE_URL',
      'https://api.lightspark.com/grid/2025-10-13',
    );
    const clientId = this.configService.get<string>('GRID_CLIENT_ID', '');
    const clientSecret = this.configService.get<string>('GRID_CLIENT_SECRET', '');
    this.enabled =
      this.configService.get<string>('GRID_ENABLED', 'false') === 'true' &&
      Boolean(clientId && clientSecret);

    this.client = axios.create({
      baseURL,
      timeout: 30000,
      auth: { username: clientId, password: clientSecret },
      headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
    });
  }

  isEnabled(): boolean {
    return this.enabled;
  }

  async createCustomer(
    payload: GridCreateCustomerPayload,
    idempotencyKey?: string,
  ): Promise<GridCustomer> {
    const { data } = await this.client.post<GridCustomer>('/customers', payload, {
      headers: idempotencyKey ? { 'Idempotency-Key': idempotencyKey } : undefined,
    });
    return data;
  }

  async getCustomer(customerId: string): Promise<GridCustomer> {
    const { data } = await this.client.get<GridCustomer>(`/customers/${customerId}`);
    return data;
  }

  async createKycLink(customerId: string, redirectUrl?: string): Promise<GridKycLink> {
    const { data } = await this.client.post<GridKycLink>(
      `/customers/${customerId}/kyc-link`,
      redirectUrl ? { redirectUrl } : {},
    );
    return data;
  }

  async listInternalAccounts(params: {
    customerId?: string;
    currency?: string;
  }): Promise<{ data: GridInternalAccount[] }> {
    const { data } = await this.client.get<{ data: GridInternalAccount[] }>(
      '/internal-accounts',
      { params },
    );
    return data;
  }

  async createExternalAccount(
    customerId: string,
    payload: GridExternalAccountPayload,
  ): Promise<GridExternalAccount> {
    const { data } = await this.client.post<GridExternalAccount>(
      `/customers/${customerId}/external-accounts`,
      payload,
    );
    return data;
  }

  async listExternalAccounts(
    customerId: string,
  ): Promise<{ data: GridExternalAccount[] }> {
    const { data } = await this.client.get<{ data: GridExternalAccount[] }>(
      `/customers/${customerId}/external-accounts`,
    );
    return data;
  }

  async deleteExternalAccount(customerId: string, accountId: string): Promise<void> {
    await this.client.delete(`/customers/${customerId}/external-accounts/${accountId}`);
  }

  async createQuote(
    payload: GridCreateQuotePayload,
    idempotencyKey?: string,
  ): Promise<GridQuote> {
    const { data } = await this.client.post<GridQuote>('/quotes', payload, {
      headers: idempotencyKey ? { 'Idempotency-Key': idempotencyKey } : undefined,
    });
    return data;
  }

  async getQuote(quoteId: string): Promise<GridQuote> {
    const { data } = await this.client.get<GridQuote>(`/quotes/${quoteId}`);
    return data;
  }

  async executeQuote(quoteId: string, idempotencyKey?: string): Promise<GridTransaction> {
    const { data } = await this.client.post<GridTransaction>(
      `/quotes/${quoteId}/execute`,
      {},
      { headers: idempotencyKey ? { 'Idempotency-Key': idempotencyKey } : undefined },
    );
    return data;
  }

  async getTransaction(transactionId: string): Promise<GridTransaction> {
    const { data } = await this.client.get<GridTransaction>(`/transactions/${transactionId}`);
    return data;
  }

  async fundInternalAccountSandbox(
    accountId: string,
    amount: string,
    currency: string,
  ): Promise<void> {
    await this.client.post(`/sandbox/internal-accounts/${accountId}/fund`, {
      amount,
      currency,
    });
  }
}
```

- [ ] **Step 4: Run tests**

```bash
npx jest src/modules/grid/providers/grid.provider.spec.ts --no-cache
```

Expected: PASS

---

### Task 5: Country & account-type mappers

**Files:**
- Create: `src/modules/grid/utils/grid-country.util.ts`
- Create: `src/modules/grid/utils/grid-account-type.mapper.ts`
- Create: `src/modules/grid/utils/grid-country.util.spec.ts`
- Create: `src/modules/grid/utils/grid-account-type.mapper.spec.ts`

- [ ] **Step 1: Write failing tests**

`grid-country.util.spec.ts`:

```typescript
import { alpha3ToAlpha2 } from './grid-country.util';

describe('alpha3ToAlpha2', () => {
  it('maps USA to US', () => expect(alpha3ToAlpha2('USA')).toBe('US'));
  it('maps MEX to MX', () => expect(alpha3ToAlpha2('MEX')).toBe('MX'));
  it('returns input if already alpha-2', () => expect(alpha3ToAlpha2('US')).toBe('US'));
});
```

`grid-account-type.mapper.spec.ts`:

```typescript
import { mapBridgeAccountTypeToGrid } from './grid-account-type.mapper';

describe('mapBridgeAccountTypeToGrid', () => {
  it('maps clabe to SPEI', () => {
    expect(mapBridgeAccountTypeToGrid('clabe')).toEqual({ accountType: 'SPEI' });
  });
  it('maps us to ACH', () => {
    expect(mapBridgeAccountTypeToGrid('us')).toEqual({ accountType: 'ACH' });
  });
});
```

- [ ] **Step 2: Run tests — FAIL**

```bash
npx jest src/modules/grid/utils --no-cache
```

- [ ] **Step 3: Implement**

`grid-country.util.ts`:

```typescript
const ALPHA3_TO_ALPHA2: Record<string, string> = {
  USA: 'US', CAN: 'CA', GBR: 'GB', MEX: 'MX', BRA: 'BR',
  IND: 'IN', DEU: 'DE', FRA: 'FR', ESP: 'ES', ITA: 'IT',
  // extend from country-id-types.config as needed
};

export function alpha3ToAlpha2(code: string): string {
  const trimmed = code?.trim().toUpperCase();
  if (!trimmed) return trimmed;
  if (trimmed.length === 2) return trimmed;
  return ALPHA3_TO_ALPHA2[trimmed] ?? trimmed.slice(0, 2);
}
```

`grid-account-type.mapper.ts`:

```typescript
export function mapBridgeAccountTypeToGrid(
  bridgeType: string,
): { accountType: string; rail?: string } {
  const map: Record<string, string> = {
    us: 'ACH', ach: 'ACH', wire: 'WIRE', sepa: 'SEPA',
    clabe: 'SPEI', pix: 'PIX', gb: 'FASTER_PAYMENTS',
  };
  const accountType = map[bridgeType.toLowerCase()];
  if (!accountType) throw new Error(`Unsupported account type: ${bridgeType}`);
  return { accountType };
}
```

- [ ] **Step 4: Run tests — PASS**

```bash
npx jest src/modules/grid/utils --no-cache
```

---

### Task 6: KYC payload mapper + GridKycHandler

**Files:**
- Create: `src/modules/grid/utils/grid-kyc-payload.mapper.ts`
- Create: `src/modules/grid/utils/grid-kyc-payload.mapper.spec.ts`
- Create: `src/modules/grid/handlers/grid-kyc.handler.ts`

- [ ] **Step 1: Write mapper test**

```typescript
import { mapKycProfileToGridCustomer } from './grid-kyc-payload.mapper';

describe('mapKycProfileToGridCustomer', () => {
  it('maps profile fields to Grid payload', () => {
    const payload = mapKycProfileToGridCustomer({
      userId: 'user_1',
      firstName: 'Jane',
      middleName: null,
      lastName: 'Doe',
      dateOfBirth: '1990-01-15',
      nationality: 'USA',
      country: 'USA',
      phone: '+15551234567',
      streetLine1: '123 Main',
      streetLine2: null,
      city: 'SF',
      state: 'CA',
      postalCode: '94105',
      taxId: '123-45-6789',
      taxIdType: 'ssn',
      user: { email: 'jane@example.com' },
    } as any);
    expect(payload.platformCustomerId).toBe('user_1');
    expect(payload.region).toBe('US');
    expect(payload.fullName).toBe('Jane Doe');
    expect(payload.identification?.type).toBe('SSN');
  });
});
```

- [ ] **Step 2: Implement mapper** (from spec Appendix B + `mapTaxIdToGridIdentification`)

- [ ] **Step 3: Implement GridKycHandler**

Key methods:
- `createOrUpdateCustomer(userId: string): Promise<GridCustomer>`
- `syncInternalAccounts(gridCustomerId: string, localCustomerId: string): Promise<void>`
- `createKycLink(userId: string): Promise<GridKycLink>`
- `getStatus(userId: string)`

Logic:
1. Load `KycProfile` + `User`
2. If `GridCustomer` exists with `gridCustomerId` → return
3. `mapKycProfileToGridCustomer` → `gridProvider.createCustomer` with idempotency key `grid-customer-${userId}-${hash}`
4. Upsert `GridCustomer`
5. `syncInternalAccounts`
6. If sandbox auto-approves or webhook pending → return

- [ ] **Step 4: Run mapper tests**

```bash
npx jest src/modules/grid/utils/grid-kyc-payload.mapper.spec.ts --no-cache
```

---

### Task 7: KYC partner integration

**Files:**
- Modify: `src/modules/kyc/config/partner-steps.config.ts`
- Modify: `src/modules/kyc/handlers/partner-submission.handler.ts`
- Modify: `src/modules/kyc/kyc.module.ts`
- Create: `src/modules/grid/grid.module.ts` (minimal shell)
- Modify: `src/app.module.ts`

- [ ] **Step 1: Add grid partner steps**

```typescript
grid: {
  partnerId: 'grid',
  steps: [
    { step: 'id_verification', required: true },
    { step: 'questionnaire', required: true },
    { step: 'grid_kyc', required: true },
  ],
},
```

- [ ] **Step 2: Add checkAndSubmitGrid to PartnerSubmissionHandler**

Inject `GridKycHandler` (use `forwardRef`).

```typescript
private async checkAndSubmitGrid(userId: string, profile: any): Promise<void> {
  if (!this.gridKycHandler.isEnabled()) return;

  const existing = await this.prisma.gridCustomer.findUnique({ where: { userId } });
  if (existing?.kycStatus === 'APPROVED') {
    await this.markGridKycApproved(profile.id);
    return;
  }

  const requiredSteps = getRequiredSteps('grid').filter((s) => s !== 'grid_kyc');
  const verifications = profile.verifications.filter((v: any) => v.partnerId === 'grid');
  const allApproved = requiredSteps.every((step) =>
    verifications.some((v: any) => v.step === step && v.status === 'approved'),
  );
  if (!allApproved) return;

  await this.gridKycHandler.createOrUpdateCustomer(userId);
}
```

Call from `checkAndSubmitIfReady` after Paytrie check.

- [ ] **Step 3: Wire modules with forwardRef**

`grid.module.ts` (minimal):

```typescript
@Module({
  imports: [ConfigModule, DatabaseModule],
  providers: [GridProvider, GridKycHandler],
  exports: [GridProvider, GridKycHandler],
})
export class GridModule {}
```

`kyc.module.ts`:

```typescript
imports: [ConfigModule, DatabaseModule, BridgeModule, PaytrieModule, forwardRef(() => GridModule)],
```

`grid.module.ts` later adds `forwardRef(() => KycModule)` only if needed — avoid circular import; GridKycHandler should NOT import KycService.

- [ ] **Step 4: Register in app.module.ts**

```typescript
import { GridModule } from './modules/grid/grid.module';
// imports: [ ..., GridModule ]
```

- [ ] **Step 5: Build**

```bash
npm run build
```

---

### Task 8: Webhook guard + handler

**Files:**
- Create: `src/modules/grid/utils/grid-webhook-verify.util.ts`
- Create: `src/modules/grid/utils/grid-webhook-verify.util.spec.ts`
- Create: `src/modules/grid/guards/grid-webhook.guard.ts`
- Create: `src/modules/grid/webhooks/grid-webhook.types.ts`
- Create: `src/modules/grid/handlers/grid-webhook.handler.ts`
- Create: `src/modules/grid/webhooks/grid-webhook.controller.ts`

- [ ] **Step 1: Write HMAC verify test**

```typescript
import * as crypto from 'crypto';
import { verifyGridWebhook } from './grid-webhook-verify.util';

it('verifies valid signature', () => {
  const secret = 'test-secret';
  const body = '{"eventType":"test"}';
  const ts = Math.floor(Date.now() / 1000).toString();
  const sig = crypto.createHmac('sha256', secret).update(`${ts}.${body}`).digest('hex');
  expect(verifyGridWebhook(Buffer.from(body), sig, ts, secret)).toBe(true);
});
```

- [ ] **Step 2: Implement verify util** (from spec §15.1)

- [ ] **Step 3: Implement GridWebhookGuard** (mirror SumsubWebhookGuard — use `request.rawBody`)

- [ ] **Step 4: Implement GridWebhookHandler**

Methods:
- `handleVerificationStatusChange`
- `handleCustomerStatusChange`
- `handleTransactionStatusChange`
- `handleIncomingPayment`

On `verification.status_change` + `APPROVED`:
- Update `GridCustomer.kycStatus`, `status = ACTIVE`
- Upsert `KycVerification` partnerId=`grid`, step=`grid_kyc`, status=`approved`

- [ ] **Step 5: GridWebhookController**

```typescript
@Controller('api/v1/grid')
export class GridWebhookController {
  @Post('webhook')
  @UseGuards(GridWebhookGuard)
  async handle(@Req() req: RawBodyRequest<Request>, @Body() body: GridWebhookEnvelope) {
    setImmediate(() => this.handler.dispatch(body));
    return { ok: true };
  }
}
```

- [ ] **Step 6: Register controller in grid.module.ts**

- [ ] **Step 7: Run verify tests**

```bash
npx jest src/modules/grid/utils/grid-webhook-verify.util.spec.ts --no-cache
```

---

### Task 9: External accounts

**Files:**
- Create: `src/modules/grid/dto/create-grid-external-account.dto.ts`
- Create: `src/modules/grid/dto/index.ts`
- Create: `src/modules/grid/handlers/grid-external-account.handler.ts`
- Modify: `src/modules/grid/grid.controller.ts` (create)
- Modify: `src/modules/grid/grid.service.ts` (create)

- [ ] **Step 1: Re-export Bridge DTOs or duplicate**

Simplest path: import `CreateExternalAccountDto` from `bridge/dto/create-external-account.dto.ts` in grid handler — same flat shape.

- [ ] **Step 2: Implement GridExternalAccountHandler**

`createExternalAccount(userId, dto)`:
1. Load `GridCustomer` where `userId`, `kycStatus=APPROVED`
2. `mapBridgeAccountTypeToGrid(dto.accountType)`
3. Build Grid payload (clabe, iban, pix, usAccount, etc.)
4. `gridProvider.createExternalAccount(gridCustomerId, payload)`
5. Persist `GridExternalAccount`

`listExternalAccounts(userId)` — map to Bridge-compatible list shape.

`deleteExternalAccount(userId, id)` — soft-delete local + Grid DELETE.

- [ ] **Step 3: GridController routes**

```typescript
@Controller('api/v1/grid')
@UseGuards(AuthGuard)
export class GridController {
  @Post('external-accounts')
  createExternalAccount(@Req() req, @Body() dto: CreateExternalAccountDto) { ... }

  @Get('external-accounts')
  listExternalAccounts(@Req() req) { ... }

  @Delete('external-accounts/:id')
  deleteExternalAccount(@Req() req, @Param('id') id: string) { ... }
}
```

- [ ] **Step 4: Manual test with sandbox** (Postman + JWT)

---

### Task 10: Funding parser

**Files:**
- Create: `src/modules/grid/utils/grid-funding.util.ts`
- Create: `src/modules/grid/utils/grid-funding.util.spec.ts`

- [ ] **Step 1: Define interface** (from spec §10.2)

- [ ] **Step 2: Write tests using spike JSON** from Task 0

If spike not done, use placeholder structure with comment linking to spike file — test with mock `paymentInstructions` object.

- [ ] **Step 3: Implement parseFundingInstructions(quote)**

Return `CryptoFundingInstruction | null`:
- `chainId: 8453`
- `tokenAddress: USDC_BASE`
- `depositAddress`
- `amountAtomic` (parseUnits)

**Fallback:** if null, handler uses `GET internal-accounts?currency=USDC` paymentInstructions.

---

### Task 11: GridPayoutHandler

**Files:**
- Create: `src/modules/grid/dto/prepare-grid-payout.dto.ts`
- Create: `src/modules/grid/dto/execute-grid-payout.dto.ts`
- Create: `src/modules/grid/dto/grid-payout-quote.dto.ts`
- Create: `src/modules/grid/handlers/grid-payout.handler.ts`
- Modify: `src/modules/grid/grid.module.ts` — import `RelayModule`

- [ ] **Step 1: Copy constants from bridge-payout.handler.ts**

`BASE_CHAIN_ID`, `USDC_BASE`, `MIN_PAYOUT_USD`, `FIXED_OUTPUT_CURRENCIES`

- [ ] **Step 2: Implement getPayoutQuote** — mirror Bridge (flat fee from `GRID_TRANSFER_DEVELOPER_FEE`)

- [ ] **Step 3: Implement preparePayout** (spec Appendix C)

- [ ] **Step 4: Implement executePayout + pollAndExecuteQuote**

Copy `waitForBaseTransactionSuccess`, `resolveRelayRequestId` patterns from `bridge-payout.handler.ts`.

- [ ] **Step 5: Implement getPayoutStatus**

- [ ] **Step 6: Implement tryExecuteQuote** (called from webhook `incoming_payment.received`)

- [ ] **Step 7: Wire controller routes**

```typescript
@Post('payout/quote')
@Post('payout/prepare')
@Post('payout/execute')
@Get('payout/:gridPayoutId/status')
```

- [ ] **Step 8: Grid KYC routes**

```typescript
@Get('kyc/status')
@Post('kyc/link')
@Get('kyc/callback')  // no AuthGuard — returns HTML deep link
```

---

### Task 12: Transaction history

**Files:**
- Modify: `src/modules/balances/handlers/transaction-history.handler.ts`

- [ ] **Step 1: Query GridTransaction for user**

Merge into existing history array with `service: 'grid'`, `type: 'PAYOUT'`.

- [ ] **Step 2: Map statuses for UI**

```typescript
function mapGridPayoutStatus(status: string): string {
  if (status === 'COMPLETED') return 'COMPLETED';
  if (['FAILED', 'QUOTE_EXPIRED', 'CANCELLED'].includes(status)) return 'FAILED';
  return 'PENDING';
}
```

---

### Task 13: GridService facade + module completion

**Files:**
- Modify: `src/modules/grid/grid.service.ts`
- Modify: `src/modules/grid/grid.module.ts`
- Create: `src/modules/grid/index.ts`

- [ ] **Step 1: GridService delegates to handlers** (mirror BridgeService)

- [ ] **Step 2: Final grid.module.ts**

```typescript
@Module({
  imports: [ConfigModule, DatabaseModule, RelayModule],
  controllers: [GridController, GridWebhookController],
  providers: [
    GridService,
    GridProvider,
    GridKycHandler,
    GridExternalAccountHandler,
    GridPayoutHandler,
    GridWebhookHandler,
    GridWebhookGuard,
  ],
  exports: [GridService, GridProvider, GridKycHandler],
})
export class GridModule {}
```

- [ ] **Step 3: Full build**

```bash
npm run build
```

---

### Task 14: E2E tests

**Files:**
- Create: `test/grid-payout.flow.e2e-spec.ts`

- [ ] **Step 1: Test KYC gating**

`POST /grid/payout/prepare` without GridCustomer → 400 `KYC_NOT_APPROVED`

- [ ] **Step 2: Test external account CRUD** (mock GridProvider in test module OR use sandbox)

Prefer injecting mock `GridProvider` in test module for CI reliability.

- [ ] **Step 3: Run e2e**

```bash
npm run test:e2e -- --testPathPattern=grid-payout
```

---

### Task 15: Documentation update

**Files:**
- Create: `docs/SEAMLESS_GRID_PAYOUT_FLOW.md` (copy from `SEAMLESS_PAYOUT_FLOW.md`, replace `/bridge` → `/grid`, `bridgePayoutId` → `gridPayoutId`)

- [ ] **Step 1: Write frontend guide**

- [ ] **Step 2: Link from spec §22**

---

## Spec Coverage Checklist

| Spec section | Task |
|--------------|------|
| §7 DB schema | Task 1 |
| §8 KYC | Tasks 6, 7, 8 |
| §9 Payout flow | Task 11 |
| §10 Crypto funding | Tasks 0, 10, 11 |
| §11 External accounts | Task 9 |
| §12 Module structure | Tasks 7, 13 |
| §13 API contract | Tasks 9, 11 |
| §14 GridProvider | Task 4 |
| §15 Webhooks | Task 8 |
| §17 Env vars | Task 2 |
| §18 Errors/idempotency | Tasks 4, 11 (use keys in provider calls) |
| §20 Testing | Tasks 14, 0 |
| §22 Frontend | Task 15 |

## Open Questions (from spec §23)

- **Q1/Q2:** Resolved by Task 0 spike before Task 10
- **Q3 BYO KYC:** Out of v1 scope — track with Lightspark separately
- **Q4 GB enum:** Confirm in Task 9 when first GB account tested

---

**Plan complete and saved to `docs/superpowers/plans/2026-06-08-lightspark-grid-payouts.md`.**

**Two execution options:**

1. **Subagent-Driven (recommended)** — Fresh subagent per task, review between tasks, fast iteration
2. **Inline Execution** — Implement tasks in this session with checkpoints

**Which approach?**
