# Seismic Integration Workflow for Enso Backend

> **Document:** Complete integration guide for migrating Enso's backend from Bridge+Sumsub to Seismic Orchestration  
> **Current Stack:** NestJS + Prisma + Sumsub (KYC) + Bridge (on/off-ramp)  
> **Target Stack:** NestJS + Prisma + Seismic Orchestration (unified)  
> **Date:** June 9, 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Architecture Analysis](#2-current-architecture-analysis)
3. [Target Architecture](#3-target-architecture)
4. [Migration Strategy](#4-migration-strategy)
5. [User Onboarding Flow (New Users)](#5-user-onboarding-flow-new-users)
6. [Existing User Migration](#6-existing-user-migration)
7. [Database Schema Changes](#7-database-schema-changes)
8. [API Implementation Guide](#8-api-implementation-guide)
9. [Webhook Implementation](#9-webhook-implementation)
10. [SDK Integration](#10-sdk-integration)
11. [Environment Variables](#11-environment-variables)
12. [Testing Checklist](#12-testing-checklist)
13. [Risk Assessment & Rollback Plan](#13-risk-assessment--rollback-plan)

---

## 1. Executive Summary

### Current State
Enso uses a **dual-provider architecture**:
- **Sumsub** → ID verification (passport, selfie) + OCR data extraction
- **Bridge** → On/off-ramp, KYC submission, TOS, wallet creation
- **Enso Backend** → Orchestrates between Sumsub and Bridge, manages KycProfile, KycVerification tables

### Target State
- **Seismic Orchestration** → Unified platform handling KYC, quotes, transfers, accounts
- **Enso Backend** → Simplified orchestration using Seismic SDK
- **Single webhook source** → All events from Seismic

### Key Benefits
| Aspect | Current | With Seismic |
|--------|---------|-------------|
| Providers | 2 (Sumsub + Bridge) | 1 (Seismic) |
| KYC flow | Custom (Sumsub SDK → Questionnaire → Bridge TOS) | Hosted KYC link or inline API |
| Webhooks | 2 sources (Sumsub + Bridge) | 1 unified source |
| Quotes | Manual per-rail | Automatic fan-out + ranking |
| Token mgmt | Sumsub tokens + Bridge API key | Single Seismic API key |
| Code complexity | High (orchestration layer) | Medium (Seismic handles more) |

---

## 2. Current Architecture Analysis

### 2.1 Current Data Flow
```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌─────────────┐
│  Sumsub SDK │────▶│   Enso       │────▶│   Bridge    │────▶│   Wallet    │
│  (ID+Selfie)│     │  Backend     │     │  (KYC+Ramp) │     │  + Accounts │
└─────────────┘     └──────────────┘     └─────────────┘     └─────────────┘
       │                    │                    │
       │  OCR data          │  KycProfile        │  BridgeCustomer
       │  (name, DOB,       │  (questionnaire    │  (kyc status,
       │   doc type, #)     │   + Sumsub merge)  │   wallet, VA)
       │                    │                    │
       ▼                    ▼                    ▼
  ┌─────────┐         ┌──────────┐         ┌──────────┐
  │ Sumsub  │         │  Prisma  │         │  Bridge  │
  │ Dashboard│         │   DB     │         │  Dashboard│
  └─────────┘         └──────────┘         └──────────┘
```

### 2.2 Current Database Tables
```
User
├── KycProfile (1:1)
│   ├── Personal info (from Sumsub OCR + questionnaire)
│   ├── sumsubApplicantId, sumsubIdDocType, sumsubGovIdDocNumber
│   └── Financial fields (occupation, sourceOfFunds, etc.)
│
├── KycVerification (1:N per partner)
│   ├── step: id_verification, questionnaire, tos, proof_of_address
│   ├── status: not_started, in_progress, approved, rejected
│   └── metadata (signedAgreementId, etc.)
│
└── BridgeCustomer (1:1)
    ├── bridgeCustomerId, kycStatus
    └── Relations: kycProfileId
```

### 2.3 Current API Endpoints
```
GET  /api/v1/kyc/status/:partnerId
GET  /api/v1/kyc/questionnaire
POST /api/v1/kyc/questionnaire
POST /api/v1/kyc/sumsub/session
GET  /api/v1/kyc/tos-link/:partnerId
POST /api/v1/kyc/tos
GET  /api/v1/kyc/tos/callback
POST /api/v1/kyc/webhook/sumsub
```

### 2.4 Current Frontend Flow
```
App Launch
    │
    ▼
GET /kyc/status/bridge
    │
    ├── overallStatus === "approved" → Show "Active" badge
    ├── overallStatus === "not_started/in_progress" → Show checklist
    └── overallStatus === "action_required" → Show failed step

Checklist:
Step 1: ID Verification (Sumsub SDK)
Step 2: Questionnaire (custom form)
Step 3: TOS (Bridge WebView)
Step 4: Proof of Address (optional)
```

---

## 3. Target Architecture

### 3.1 Target Data Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                         ENSE BACKEND                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Seismic   │    │   User      │    │   Order     │         │
│  │   Provider  │    │   Service   │    │   Service   │         │
│  └──────┬──────┘    └─────────────┘    └─────────────┘         │
│         │                                                        │
│         │  API Key Auth                                          │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────┐         │
│  │           SEISMIC ORCHESTRATION API                  │         │
│  │  • KYC (hosted or inline)                            │         │
│  │  • Quotes (fan-out across rails)                     │         │
│  │  • Accounts (fiat + crypto)                          │         │
│  │  • Transfers (on/off-ramp)                           │         │
│  │  • Webhooks (unified events)                         │         │
│  └─────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Target Database Tables
```
User
├── SeismicCustomer (1:1)
│   ├── seismicCustomerId (UUID)
│   ├── kycStatus (not_started, submitted, under_review, approved, rejected)
│   ├── kycLinkUrl (hosted KYC URL)
│   ├── kycLinkToken
│   └── profileSnapshot (JSON - last known profile from Seismic)
│
├── SeismicAccount[] (1:N)
│   ├── seismicAccountId
│   ├── kind (fiat/crypto)
│   ├── currency
│   ├── rails[]
│   └── status
│
└── Order[] (1:N)
    ├── seismicOrderId
    ├── status
    ├── amount, currency
    └── type (onramp/offramp/swap)

KycProfile (保留 for historical data, new data goes to Seismic)
```

### 3.3 Target API Endpoints
```
# Seismic module (NEW)
POST /api/v1/seismic/customers              # Create customer on Seismic
GET  /api/v1/seismic/customers/:id/status   # Get KYC status
POST /api/v1/seismic/customers/:id/kyc      # Submit KYC profile
POST /api/v1/seismic/quotes                 # Request quotes
POST /api/v1/seismic/quotes/:id/accept      # Accept quote
POST /api/v1/seismic/accounts               # Create account

# Webhook (NEW - replaces Sumsub + Bridge webhooks)
POST /api/v1/webhooks/seismic               # Unified webhook handler

# Legacy KYC module (DEPRECATED - keep for migration)
# All /api/v1/kyc/* endpoints → Mark deprecated, redirect to Seismic
```

---

## 4. Migration Strategy

### 4.1 Migration Phases

```
Phase 1: Parallel Setup (Week 1-2)
├── Set up Seismic org + KYB
├── Mint API keys
├── Install seismic-orchestration SDK
├── Create Seismic module in Enso backend
└── Implement webhook handler

Phase 2: New Users → Seismic (Week 2-3)
├── New user onboarding flows through Seismic
├── Keep Sumsub+Bridge for existing in-progress users
├── Dual-write status for visibility
└── Monitor Seismic webhooks

Phase 3: Existing User Migration (Week 3-4)
├── ACTIVE BridgeCustomer users → Option A or B
├── In-progress users → Re-route through Seismic
└── Clean up legacy data

Phase 4: Cleanup (Week 4-5)
├── Remove Sumsub integration
├── Remove direct Bridge integration
├── Deprecate /api/v1/kyc/* endpoints
└── Full cutover to Seismic
```

### 4.2 Migration Options for Existing ACTIVE Users

#### Option A: Re-KYC through Seismic (Safest)
- Notify users they need to complete quick re-verification
- Seismic hosted KYC link handles everything
- Fresh, clean data in Seismic

#### Option B: Data Migration (If Seismic Supports)
- Export KYC data from Sumsub/Bridge
- Import into Seismic via API
- Requires Seismic support for pre-approved KYC
- **⚠️ Confirm with Seismic team before choosing this**

#### Option C: Maintain Legacy (Hybrid)
- Keep existing Bridge customers on Bridge
- New customers go through Seismic
- Long-term: gradual migration

---

## 5. User Onboarding Flow (New Users)

### 5.1 Flow Overview
```
User taps "Get Bank Account" (EUR/USD)
    │
    ▼
[Enso Backend] Create Seismic Customer
    │
    ▼
POST /api/v1/seismic/customers
    │
    ├── Creates customer on Seismic
    ├── Returns: customerId + kycLink.url
    └── Saves seismicCustomerId to Enso DB
    │
    ▼
[Frontend] Redirect to Hosted KYC Link
    │
    ▼
User completes KYC in Seismic-hosted UI
(ID upload, selfie, questionnaire, TOS)
    │
    ▼
Seismic processes KYC
    │
    ▼
Webhook: customer.kyc.status_changed
    │
    ├── status === "approved"
    │       ├── Create accounts (fiat + crypto)
    │       └── Notify user: "Ready!"
    │
    ├── status === "rejected"
    │       └── Notify user: "Please retry"
    │
    └── status === "under_review"
            └── Show: "Processing..."
```

### 5.2 Detailed Flow

#### Step 1: Create Customer
```typescript
// POST /api/v1/seismic/customers
async function createSeismicCustomer(userId: string) {
  const user = await prisma.user.findUnique({ where: { id: userId } });
  
  // Create on Seismic
  const customer = await seismicApi.customers.create({
    type: "individual",
    profile: {
      identity: {
        first_name: user.firstName || "",
        last_name: user.lastName || "",
        date_of_birth: user.dateOfBirth || "",
      },
      contact: {
        email: user.email,
        phone: user.phone || undefined,
      },
      addresses: {
        residential: {
          line1: user.streetLine1 || "",
          line2: user.streetLine2 || undefined,
          city: user.city || "",
          state: user.state || undefined,
          postal_code: user.postalCode || "",
          country: user.country || "",
        }
      }
    }
  });
  
  // Save to Enso DB
  await prisma.seismicCustomer.create({
    data: {
      userId,
      seismicCustomerId: customer.id,
      kycStatus: customer.kyc.status,
      kycLinkUrl: customer.kyc.link.url,
      kycLinkToken: customer.kyc.link.token,
    }
  });
  
  return {
    customerId: customer.id,
    kycLinkUrl: customer.kyc.link.url,
  };
}
```

#### Step 2: User Completes Hosted KYC
- Frontend redirects to `kycLinkUrl`
- Seismic handles: ID verification, selfie, questionnaire, TOS
- No Sumsub SDK needed
- No custom questionnaire form needed
- No Bridge TOS WebView needed

#### Step 3: Webhook Handler
```typescript
// POST /api/v1/webhooks/seismic
async function handleSeismicWebhook(payload: WebhookPayload) {
  if (payload.event_type === "customer.kyc.status_changed") {
    const { customer_id, status, kyc_profile } = payload.data;
    
    // Find Enso user
    const seismicCustomer = await prisma.seismicCustomer.findUnique({
      where: { seismicCustomerId: customer_id },
      include: { user: true }
    });
    
    if (!seismicCustomer) return;
    
    // Update status
    await prisma.seismicCustomer.update({
      where: { seismicCustomerId: customer_id },
      data: {
        kycStatus: status,
        profileSnapshot: kyc_profile,
      }
    });
    
    if (status === "approved") {
      // Create accounts
      await createDefaultAccounts(seismicCustomer.userId, customer_id);
      
      // Notify user
      await notifyUser(seismicCustomer.userId, {
        type: "KYC_APPROVED",
        message: "Your verification is complete!"
      });
    } else if (status === "rejected") {
      await notifyUser(seismicCustomer.userId, {
        type: "KYC_REJECTED",
        message: "Verification failed. Please try again."
      });
    }
  }
}
```

#### Step 4: Create Accounts (Post-KYC Approval)
```typescript
async function createDefaultAccounts(userId: string, customerId: string) {
  // Create USD virtual account (ACH + Wire)
  const usdAccount = await seismicApi.customers.createFiatVirtualAccount(
    customerId,
    {
      currency: "USD",
      rails: ["ach", "wire"],
      schema: "us_bank"
    }
  );
  
  // Create USDC virtual account (Base chain)
  const usdcAccount = await seismicApi.customers.createCryptoVirtualAccount(
    customerId,
    {
      currency: "USDC",
      rails: ["base"],
      schema: "evm"
    }
  );
  
  // Save to Enso DB
  await prisma.seismicAccount.createMany({
    data: [
      {
        userId,
        seismicAccountId: usdAccount.id,
        kind: "fiat",
        currency: "USD",
        rails: ["ach", "wire"],
        status: usdAccount.status,
      },
      {
        userId,
        seismicAccountId: usdcAccount.id,
        kind: "crypto",
        currency: "USDC",
        rails: ["base"],
        status: usdcAccount.status,
      }
    ]
  });
  
  return { usdAccount, usdcAccount };
}
```

---

## 6. Existing User Migration

### 6.1 Migration Categories

| User Category | Count (Est.) | Strategy | Effort |
|--------------|-------------|----------|--------|
| **ACTIVE** (BridgeCustomer.status = ACTIVE) | TBD | Option A (re-KYC) or Option C (keep on Bridge) | Medium |
| **KYC In-Progress** (Sumsub done, pending Bridge) | TBD | Re-route through Seismic | Low |
| **KYC Not Started** | TBD | Direct to Seismic | None |
| **Rejected** | TBD | Re-try through Seismic | Low |

### 6.2 ACTIVE User Migration

#### Option A: Re-KYC (Recommended)
```typescript
// For each ACTIVE user:
async function migrateActiveUser(userId: string) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    include: { kycProfile: true, bridgeCustomer: true }
  });
  
  // 1. Create Seismic customer
  const { customerId, kycLinkUrl } = await createSeismicCustomer(userId);
  
  // 2. Pre-fill profile from existing KycProfile
  if (user.kycProfile) {
    await seismicApi.customers.updateKycProfile(customerId, {
      identity: {
        first_name: user.kycProfile.firstName,
        last_name: user.kycProfile.lastName,
        date_of_birth: user.kycProfile.dateOfBirth,
        nationality: user.kycProfile.nationality,
      },
      contact: {
        phone: user.kycProfile.phone,
      },
      addresses: {
        residential: {
          line1: user.kycProfile.streetLine1,
          line2: user.kycProfile.streetLine2,
          city: user.kycProfile.city,
          state: user.kycProfile.state,
          postal_code: user.kycProfile.postalCode,
          country: user.kycProfile.country,
        }
      }
    });
  }
  
  // 3. Send notification to user
  await notifyUser(userId, {
    type: "MIGRATION_REKYC_REQUIRED",
    message: "Please complete quick re-verification",
    actionUrl: kycLinkUrl,
  });
  
  // 4. When webhook comes back approved → create accounts
}
```

#### Option B: Grandfather (Keep on Bridge)
```typescript
// Mark these users as "legacy_bridge" 
// Keep existing Bridge integration running for them
// Only new users go through Seismic

// In your KYC status check:
async function getKycStatus(userId: string) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    include: { seismicCustomer: true, bridgeCustomer: true }
  });
  
  if (user.seismicCustomer) {
    // Check Seismic status
    return getSeismicKycStatus(user.seismicCustomer);
  } else if (user.bridgeCustomer?.status === "ACTIVE") {
    // Legacy Bridge user
    return { status: "approved", source: "bridge_legacy" };
  }
}
```

### 6.3 Data Mapping (Enso → Seismic)

| Enso Field | Seismic Field | Notes |
|-----------|--------------|-------|
| `User.id` | — | Enso's internal ID (keep) |
| `KycProfile.firstName` | `profile.identity.first_name` | |
| `KycProfile.lastName` | `profile.identity.last_name` | |
| `KycProfile.middleName` | — | Seismic may not have middle name |
| `KycProfile.dateOfBirth` | `profile.identity.date_of_birth` | Format: YYYY-MM-DD |
| `KycProfile.nationality` | `profile.identity.nationality` | ISO alpha-3 |
| `KycProfile.phone` | `profile.contact.phone` | E.164 format |
| `KycProfile.phoneCountryCode` | — | Part of phone E.164 |
| `KycProfile.streetLine1` | `profile.addresses.residential.line1` | |
| `KycProfile.streetLine2` | `profile.addresses.residential.line2` | |
| `KycProfile.city` | `profile.addresses.residential.city` | |
| `KycProfile.state` | `profile.addresses.residential.state` | |
| `KycProfile.postalCode` | `profile.addresses.residential.postal_code` | |
| `KycProfile.country` | `profile.addresses.residential.country` | ISO alpha-3 |
| `KycProfile.occupation` | `profile.employment.occupation` | SOC code or text |
| `KycProfile.employmentStatus` | `profile.employment.employment_status` | |
| `KycProfile.sourceOfFunds` | `profile.risk.source_of_funds` | |
| `KycProfile.accountPurpose` | `profile.risk.account_purpose` | |
| `KycProfile.expectedMonthlyVolume` | `profile.volumes.expected_monthly_volume` | |
| `KycProfile.actingAsIntermediary` | `profile.attestations` | Part of attestations |
| `KycProfile.taxId` | `profile.jurisdictions.tax_id` | |
| `KycProfile.taxIdCountry` | `profile.jurisdictions.tax_id_country` | |
| `KycProfile.taxIdType` | — | Auto-resolved from country |
| `KycProfile.sumsubApplicantId` | — | Not needed with Seismic |
| `KycProfile.sumsubIdDocType` | — | Handled by Seismic KYC |
| `KycProfile.sumsubGovIdDocNumber` | — | Handled by Seismic KYC |
| `sumsubVerifiedAt` | — | Not needed with Seismic |

---

## 7. Database Schema Changes

### 7.1 New Tables

```prisma
// prisma/schema.prisma

model SeismicCustomer {
  id                String   @id @default(cuid())
  userId            String   @unique
  user              User     @relation(fields: [userId], references: [id])
  
  seismicCustomerId String   @unique  // UUID from Seismic
  kycStatus         String   // not_started, submitted, under_review, approved, rejected
  kycLinkUrl        String?
  kycLinkToken      String?
  profileSnapshot   Json?    // Last known KYC profile from Seismic
  
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
}

model SeismicAccount {
  id              String   @id @default(cuid())
  userId          String
  user            User     @relation(fields: [userId], references: [id])
  
  seismicAccountId String  @unique
  kind            String   // fiat, crypto
  shape           String?  // us, iban, evm, sol, etc.
  currency        String
  rails           String[] // ach, wire, base, etc.
  status          String   // active, pending, closed
  details         Json?    // Account-specific details
  
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}

model SeismicOrder {
  id            String   @id @default(cuid())
  userId        String
  user          User     @relation(fields: [userId], references: [id])
  
  seismicOrderId String  @unique
  status         String  // awaiting_funds, pending, processing, completed, failed, canceled
  type           String  // onramp, offramp, swap
  
  sourceCurrency      String
  sourceAmount        String
  sourceRail          String?
  destinationCurrency String
  destinationAmount   String?
  destinationRail     String?
  
  fees            Json?
  rate            String?
  
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}
```

### 7.2 Modified Tables

```prisma
model User {
  // ... existing fields ...
  
  // NEW: Seismic relations
  seismicCustomer  SeismicCustomer?
  seismicAccounts  SeismicAccount[]
  seismicOrders    SeismicOrder[]
  
  // KEEP (for migration period):
  kycProfile       KycProfile?
  bridgeCustomer   BridgeCustomer?
}
```

### 7.3 Migration Script

```typescript
// scripts/migrate-to-seismic.ts
async function migrateDatabase() {
  // 1. Create new tables
  await prisma.$executeRaw`
    CREATE TABLE IF NOT EXISTS "SeismicCustomer" (
      id TEXT PRIMARY KEY,
      "userId" TEXT UNIQUE REFERENCES "User"(id),
      "seismicCustomerId" TEXT UNIQUE,
      "kycStatus" TEXT NOT NULL DEFAULT 'not_started',
      "kycLinkUrl" TEXT,
      "kycLinkToken" TEXT,
      "profileSnapshot" JSONB,
      "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      "updatedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    CREATE TABLE IF NOT EXISTS "SeismicAccount" (
      id TEXT PRIMARY KEY,
      "userId" TEXT REFERENCES "User"(id),
      "seismicAccountId" TEXT UNIQUE,
      kind TEXT NOT NULL,
      shape TEXT,
      currency TEXT NOT NULL,
      rails TEXT[],
      status TEXT NOT NULL,
      details JSONB,
      "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      "updatedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    CREATE TABLE IF NOT EXISTS "SeismicOrder" (
      id TEXT PRIMARY KEY,
      "userId" TEXT REFERENCES "User"(id),
      "seismicOrderId" TEXT UNIQUE,
      status TEXT NOT NULL,
      type TEXT NOT NULL,
      "sourceCurrency" TEXT NOT NULL,
      "sourceAmount" TEXT NOT NULL,
      "sourceRail" TEXT,
      "destinationCurrency" TEXT NOT NULL,
      "destinationAmount" TEXT,
      "destinationRail" TEXT,
      fees JSONB,
      rate TEXT,
      "createdAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      "updatedAt" TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
  `;
  
  console.log("Migration complete");
}
```

---

## 8. API Implementation Guide

### 8.1 Module Structure

```
src/modules/seismic/
├── seismic.module.ts
├── seismic.controller.ts
├── seismic.service.ts
├── seismic.provider.ts
├── dto/
│   ├── create-customer.dto.ts
│   ├── update-kyc.dto.ts
│   ├── create-quote.dto.ts
│   └── create-account.dto.ts
├── handlers/
│   ├── kyc-status.handler.ts
│   ├── order-status.handler.ts
│   └── account-event.handler.ts
└── utils/
    ├── profile-mapper.ts
    └── status-mapper.ts
```

### 8.2 SeismicProvider

```typescript
// src/modules/seismic/seismic.provider.ts
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { SeismicOrchestrationClient } from "seismic-orchestration";

@Injectable()
export class SeismicProvider {
  private api: ReturnType<typeof SeismicOrchestrationClient.sandbox>;
  
  constructor(private config: ConfigService) {
    const apiKey = this.config.get<string>("SEISMIC_API_KEY");
    const isProduction = this.config.get<string>("NODE_ENV") === "production";
    
    if (isProduction) {
      this.api = SeismicOrchestrationClient.production({ apiKey });
    } else {
      this.api = SeismicOrchestrationClient.sandbox({ apiKey });
    }
  }
  
  getApi() {
    return this.api;
  }
}
```

### 8.3 SeismicService

```typescript
// src/modules/seismic/seismic.service.ts
import { Injectable } from "@nestjs/common";
import { SeismicProvider } from "./seismic.provider";
import { PrismaService } from "../prisma/prisma.service";

@Injectable()
export class SeismicService {
  constructor(
    private seismic: SeismicProvider,
    private prisma: PrismaService
  ) {}
  
  async createCustomer(userId: string, profile: KycProfileInput) {
    const api = this.seismic.getApi();
    
    const customer = await api.customers.create({
      type: "individual",
      profile: this.mapProfile(profile)
    });
    
    await this.prisma.seismicCustomer.create({
      data: {
        userId,
        seismicCustomerId: customer.id,
        kycStatus: customer.kyc.status,
        kycLinkUrl: customer.kyc.link.url,
        kycLinkToken: customer.kyc.link.token,
      }
    });
    
    return customer;
  }
  
  async getQuotes(userId: string, request: QuoteRequest) {
    const seismicCustomer = await this.prisma.seismicCustomer.findUnique({
      where: { userId }
    });
    
    if (!seismicCustomer) {
      throw new Error("Customer not found on Seismic");
    }
    
    const api = this.seismic.getApi();
    
    return api.quotes.create({
      end_customer_id: seismicCustomer.seismicCustomerId,
      end_customer_type: "individual",
      amount: request.amount,
      source: request.source,
      destination: request.destination
    }, { wait: true });
  }
  
  async acceptQuote(userId: string, snapshotId: string) {
    const api = this.seismic.getApi();
    const order = await api.quotes.acceptBest(snapshotId);
    
    await this.prisma.seismicOrder.create({
      data: {
        userId,
        seismicOrderId: order.id,
        status: order.status,
        type: this.inferOrderType(order),
        sourceCurrency: order.source.currency,
        sourceAmount: order.source.amount,
        sourceRail: order.source.rail,
        destinationCurrency: order.destination.currency,
        destinationAmount: order.destination.amount,
        destinationRail: order.destination.rail,
        fees: order.fees,
        rate: order.rate,
      }
    });
    
    return order;
  }
  
  private mapProfile(input: KycProfileInput) {
    return {
      identity: {
        first_name: input.firstName,
        last_name: input.lastName,
        date_of_birth: input.dateOfBirth,
        nationality: input.nationality,
      },
      contact: {
        email: input.email,
        phone: input.phone,
      },
      addresses: {
        residential: {
          line1: input.streetLine1,
          line2: input.streetLine2,
          city: input.city,
          state: input.state,
          postal_code: input.postalCode,
          country: input.country,
        }
      },
      ...(input.occupation && {
        employment: {
          occupation: input.occupation,
          employment_status: input.employmentStatus,
        }
      }),
      ...(input.sourceOfFunds && {
        risk: {
          source_of_funds: input.sourceOfFunds,
          account_purpose: input.accountPurpose,
        }
      }),
      ...(input.expectedMonthlyVolume && {
        volumes: {
          expected_monthly_volume: input.expectedMonthlyVolume,
        }
      }),
      ...(input.taxId && {
        jurisdictions: {
          tax_id: input.taxId,
          tax_id_country: input.taxIdCountry || input.country,
        }
      }),
    };
  }
  
  private inferOrderType(order: any): string {
    const sourceFiat = ["USD", "EUR", "GBP"].includes(order.source.currency);
    const destFiat = ["USD", "EUR", "GBP"].includes(order.destination.currency);
    
    if (sourceFiat && !destFiat) return "onramp";
    if (!sourceFiat && destFiat) return "offramp";
    return "swap";
  }
}
```

### 8.4 Webhook Controller

```typescript
// src/modules/seismic/seismic-webhook.controller.ts
import { Controller, Post, Body, Headers, BadRequestException } from "@nestjs/common";
import { Webhook } from "svix";
import { ConfigService } from "@nestjs/config";
import { SeismicWebhookService } from "./seismic-webhook.service";

@Controller("api/v1/webhooks/seismic")
export class SeismicWebhookController {
  private webhookSecret: string;
  
  constructor(
    private config: ConfigService,
    private webhookService: SeismicWebhookService
  ) {
    this.webhookSecret = this.config.get<string>("SEISMIC_WEBHOOK_SECRET");
  }
  
  @Post()
  async handleWebhook(
    @Body() payload: any,
    @Headers() headers: Record<string, string>
  ) {
    const wh = new Webhook(this.webhookSecret);
    
    let event;
    try {
      event = wh.verify(payload, headers);
    } catch (err) {
      throw new BadRequestException("Invalid webhook signature");
    }
    
    await this.webhookService.handleEvent(event);
    
    return { received: true };
  }
}
```

---

## 9. Webhook Implementation

### 9.1 Svix Setup
```bash
npm install svix
```

### 9.2 Webhook Service
```typescript
// src/modules/seismic/seismic-webhook.service.ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "../prisma/prisma.service";
import { NotificationService } from "../notification/notification.service";

@Injectable()
export class SeismicWebhookService {
  constructor(
    private prisma: PrismaService,
    private notifications: NotificationService
  ) {}
  
  async handleEvent(event: any) {
    switch (event.event_type) {
      case "customer.kyc.status_changed":
        await this.handleKycStatusChanged(event.data);
        break;
      case "account.created":
      case "account.updated":
        await this.handleAccountEvent(event.data);
        break;
      case "order.status_changed":
        await this.handleOrderStatusChanged(event.data);
        break;
      default:
        console.warn(`Unhandled webhook event: ${event.event_type}`);
    }
  }
  
  private async handleKycStatusChanged(data: any) {
    const { customer_id, status, kyc_profile } = data;
    
    const seismicCustomer = await this.prisma.seismicCustomer.findUnique({
      where: { seismicCustomerId: customer_id }
    });
    
    if (!seismicCustomer) {
      console.error(`Unknown Seismic customer: ${customer_id}`);
      return;
    }
    
    await this.prisma.seismicCustomer.update({
      where: { seismicCustomerId: customer_id },
      data: {
        kycStatus: status,
        profileSnapshot: kyc_profile || undefined,
      }
    });
    
    if (status === "approved") {
      // Create default accounts
      await this.createDefaultAccounts(seismicCustomer.userId, customer_id);
      
      // Notify user
      await this.notifications.send(seismicCustomer.userId, {
        type: "KYC_APPROVED",
        title: "Verification Complete",
        body: "Your identity verification is complete! You can now start transacting."
      });
    } else if (status === "rejected") {
      await this.notifications.send(seismicCustomer.userId, {
        type: "KYC_REJECTED",
        title: "Verification Failed",
        body: "Your verification was not approved. Please try again."
      });
    }
  }
  
  private async handleAccountEvent(data: any) {
    const { account_id, customer_id, account } = data;
    
    const seismicCustomer = await this.prisma.seismicCustomer.findUnique({
      where: { seismicCustomerId: customer_id }
    });
    
    if (!seismicCustomer) return;
    
    // Upsert account
    await this.prisma.seismicAccount.upsert({
      where: { seismicAccountId: account_id },
      create: {
        userId: seismicCustomer.userId,
        seismicAccountId: account_id,
        kind: account.kind,
        shape: account.shape,
        currency: account.currency,
        rails: account.rails,
        status: account.status,
        details: account.details,
      },
      update: {
        status: account.status,
        details: account.details,
      }
    });
  }
  
  private async handleOrderStatusChanged(data: any) {
    const { order_id, status, order } = data;
    
    await this.prisma.seismicOrder.updateMany({
      where: { seismicOrderId: order_id },
      data: { status }
    });
    
    const seismicOrder = await this.prisma.seismicOrder.findUnique({
      where: { seismicOrderId: order_id }
    });
    
    if (seismicOrder) {
      const title = status === "completed" ? "Transfer Complete" :
                    status === "failed" ? "Transfer Failed" : "Transfer Update";
      
      await this.notifications.send(seismicOrder.userId, {
        type: `ORDER_${status.toUpperCase()}`,
        title,
        body: `Your ${order.source.currency} → ${order.destination.currency} transfer is ${status}.`
      });
    }
  }
  
  private async createDefaultAccounts(userId: string, customerId: string) {
    const { SeismicOrchestrationClient } = await import("seismic-orchestration");
    const api = SeismicOrchestrationClient.sandbox({
      apiKey: process.env.SEISMIC_API_KEY
    });
    
    // Create USD account
    try {
      const usdAccount = await api.customers.createFiatVirtualAccount(
        customerId,
        { currency: "USD", rails: ["ach", "wire"], schema: "us_bank" }
      );
      
      await this.prisma.seismicAccount.create({
        data: {
          userId,
          seismicAccountId: usdAccount.id,
          kind: "fiat",
          currency: "USD",
          rails: ["ach", "wire"],
          status: usdAccount.status,
        }
      });
    } catch (err) {
      console.error("Failed to create USD account:", err);
    }
    
    // Create USDC account
    try {
      const usdcAccount = await api.customers.createCryptoVirtualAccount(
        customerId,
        { currency: "USDC", rails: ["base"], schema: "evm" }
      );
      
      await this.prisma.seismicAccount.create({
        data: {
          userId,
          seismicAccountId: usdcAccount.id,
          kind: "crypto",
          currency: "USDC",
          rails: ["base"],
          status: usdcAccount.status,
        }
      });
    } catch (err) {
      console.error("Failed to create USDC account:", err);
    }
  }
}
```

---

## 10. SDK Integration

### 10.1 Install
```bash
npm install seismic-orchestration svix
```

### 10.2 Module Registration
```typescript
// src/modules/seismic/seismic.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { SeismicProvider } from "./seismic.provider";
import { SeismicService } from "./seismic.service";
import { SeismicController } from "./seismic.controller";
import { SeismicWebhookController } from "./seismic-webhook.controller";
import { SeismicWebhookService } from "./seismic-webhook.service";

@Module({
  imports: [ConfigModule],
  controllers: [SeismicController, SeismicWebhookController],
  providers: [SeismicProvider, SeismicService, SeismicWebhookService],
  exports: [SeismicService],
})
export class SeismicModule {}
```

### 10.3 App Module
```typescript
// src/app.module.ts
import { Module } from "@nestjs/common";
import { SeismicModule } from "./modules/seismic/seismic.module";
// ... existing imports

@Module({
  imports: [
    // ... existing modules
    SeismicModule,  // NEW
  ],
})
export class AppModule {}
```

---

## 11. Environment Variables

### 11.1 New Variables
```bash
# Seismic Integration (NEW)
SEISMIC_API_KEY=seismic_sandbox_abc123...
SEISMIC_BASE_URL=https://orchestration-sandbox.seismictest.net/api
SEISMIC_WEBHOOK_SECRET=whsec_...

# Seismic KYB (your org's verification)
SEISMIC_ORG_KYB_STATUS=pending  # Track your org's KYB status
```

### 11.2 Deprecated Variables (Remove after migration)
```bash
# Sumsub (DEPRECATED - remove after migration)
SUMSUB_APP_TOKEN=sbx:...
SUMSUB_SECRET_KEY=...
SUMSUB_LEVEL_NAME=basic-kyc-level
SUMSUB_BASE_URL=https://api.sumsub.com
SUMSUB_WEBHOOK_SECRET=...

# Bridge (DEPRECATED - remove after migration)
BRIDGE_API_KEY=...
BRIDGE_API_URL=...
```

### 11.3 Validation Schema
```typescript
// src/config/validation.schema.ts
export const validationSchema = Joi.object({
  // ... existing vars
  
  // Seismic (NEW)
  SEISMIC_API_KEY: Joi.string().required(),
  SEISMIC_BASE_URL: Joi.string().uri().default(
    "https://orchestration-sandbox.seismictest.net/api"
  ),
  SEISMIC_WEBHOOK_SECRET: Joi.string().required(),
  
  // Legacy (DEPRECATED but keep during migration)
  SUMSUB_APP_TOKEN: Joi.string(),
  SUMSUB_SECRET_KEY: Joi.string(),
  BRIDGE_API_KEY: Joi.string(),
});
```

---

## 12. Testing Checklist

### 12.1 Unit Tests
- [ ] SeismicProvider initialization (sandbox vs production)
- [ ] Profile mapper (Enso → Seismic format)
- [ ] Status mapper (Seismic → Enso format)
- [ ] Webhook signature verification
- [ ] Error handling for each Seismic error code

### 12.2 Integration Tests
- [ ] Create customer on Seismic
- [ ] Update KYC profile
- [ ] Submit KYC
- [ ] Poll KYC status
- [ ] Create fiat virtual account
- [ ] Create crypto virtual account
- [ ] Request quotes (fan-out)
- [ ] Accept quote → create order
- [ ] Receive and process KYC webhook
- [ ] Receive and process order webhook
- [ ] Receive and process account webhook

### 12.3 End-to-End Tests
- [ ] New user: Sign up → Create Seismic customer → Complete KYC → Accounts created
- [ ] Onramp: Request quote (USD → USDC) → Accept → Funds received → Completed
- [ ] Offramp: Request quote (USDC → EUR) → Accept → Completed
- [ ] Swap: Request quote (USDC → USDC different chain) → Accept → Completed
- [ ] KYC rejection: Submit bad docs → Rejected → Retry → Approved

### 12.4 Migration Tests
- [ ] Existing ACTIVE user: Re-KYC flow works
- [ ] Existing in-progress user: Re-route works
- [ ] Data migration: Profile fields correctly mapped
- [ ] Legacy endpoint deprecation: Returns appropriate warning

### 12.5 Regression Tests
- [ ] Existing users on Bridge still work (during migration)
- [ ] No data loss in KycProfile table
- [ ] Frontend checklist screen displays correct status

---

## 13. Risk Assessment & Rollback Plan

### 13.1 Risk Matrix

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Seismic API downtime | Low | High | Keep Bridge fallback during migration |
| KYC rejection rate increase | Medium | Medium | A/B test with small user group first |
| Data mapping errors | Medium | High | Thorough testing + data validation |
| User confusion (re-KYC) | Medium | Low | Clear in-app messaging |
| Seismic feature gaps | Medium | High | Maintain dual integration during evaluation |
| Webhook delivery failures | Low | Medium | Svix retry + manual fallback |

### 13.2 Rollback Plan

```
IF critical issue detected:
  1. Disable Seismic customer creation (feature flag)
  2. Route new users back to Sumsub + Bridge flow
  3. Keep existing Seismic users on Seismic
  4. Fix issue
  5. Re-enable with smaller rollout
```

### 13.3 Feature Flags
```typescript
// src/config/features.ts
export const FEATURES = {
  // Enable Seismic for new users
  SEISMIC_ONBOARDING: process.env.FF_SEISMIC_ONBOARDING === "true",
  
  // Enable Seismic quotes for existing users
  SEISMIC_QUOTES: process.env.FF_SEISMIC_QUOTES === "true",
  
  // Enable Seismic webhooks
  SEISMIC_WEBHOOKS: process.env.FF_SEISMIC_WEBHOOKS === "true",
  
  // Keep legacy integration running
  LEGACY_BRIDGE: process.env.FF_LEGACY_BRIDGE !== "false",
  LEGACY_SUMSUB: process.env.FF_LEGACY_SUMSUB !== "false",
};
```

---

## Appendix A: File Structure (Target)

```
src/
├── modules/
│   ├── seismic/                          # NEW MODULE
│   │   ├── seismic.module.ts
│   │   ├── seismic.controller.ts         # API endpoints
│   │   ├── seismic.service.ts            # Business logic
│   │   ├── seismic.provider.ts           # SDK client
│   │   ├── dto/
│   │   │   ├── create-customer.dto.ts
│   │   │   ├── update-kyc.dto.ts
│   │   │   ├── create-quote.dto.ts
│   │   │   └── create-account.dto.ts
│   │   ├── handlers/
│   │   │   ├── kyc-status.handler.ts
│   │   │   ├── order-status.handler.ts
│   │   │   └── account-event.handler.ts
│   │   └── utils/
│   │       ├── profile-mapper.ts
│   │       └── status-mapper.ts
│   │
│   ├── kyc/                              # EXISTING (DEPRECATED)
│   │   ├── ...                           # Keep for migration
│   │
│   ├── bridge/                           # EXISTING (DEPRECATED)
│   │   ├── ...                           # Keep for migration
│   │
│   └── ...
│
├── prisma/
│   ├── schema.prisma                     # Add Seismic tables
│   └── migrations/
│       └── 20260609_add_seismic/
│
└── config/
    └── validation.schema.ts              # Add Seismic env vars
```

## Appendix B: API Endpoint Summary

### New Endpoints (Seismic)
| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/seismic/customers` | Create Seismic customer |
| `GET` | `/api/v1/seismic/customers/:id` | Get customer + KYC status |
| `PUT` | `/api/v1/seismic/customers/:id/kyc` | Update KYC profile |
| `POST` | `/api/v1/seismic/customers/:id/kyc/submit` | Submit KYC |
| `POST` | `/api/v1/seismic/quotes` | Request quotes |
| `POST` | `/api/v1/seismic/quotes/:id/accept` | Accept quote |
| `POST` | `/api/v1/seismic/accounts` | Create account |
| `POST` | `/api/v1/webhooks/seismic` | Seismic webhook handler |

### Deprecated Endpoints (Keep during migration)
| Method | Endpoint | Status |
|--------|----------|--------|
| `GET` | `/api/v1/kyc/status/:partnerId` | Deprecated → Use `/seismic/customers/:id` |
| `GET` | `/api/v1/kyc/questionnaire` | Deprecated → Seismic handles this |
| `POST` | `/api/v1/kyc/questionnaire` | Deprecated → Seismic handles this |
| `POST` | `/api/v1/kyc/sumsub/session` | Deprecated → No Sumsub needed |
| `GET` | `/api/v1/kyc/tos-link/:partnerId` | Deprecated → Seismic handles TOS |
| `POST` | `/api/v1/kyc/tos` | Deprecated → Seismic handles TOS |
| `GET` | `/api/v1/kyc/tos/callback` | Deprecated → Seismic handles TOS |
| `POST` | `/api/v1/kyc/webhook/sumsub` | Deprecated → Use `/webhooks/seismic` |

## Appendix C: Timeline Estimate

| Phase | Duration | Key Deliverables |
|-------|----------|-----------------|
| **Phase 0: Setup** | 1-2 days | Seismic org, KYB, API keys, SDK install |
| **Phase 1: Core Integration** | 3-5 days | Seismic module, customer creation, webhook handler |
| **Phase 2: New User Flow** | 3-5 days | End-to-end onboarding, KYC, account creation |
| **Phase 3: Quotes & Transfers** | 3-5 days | Quote requests, acceptance, order tracking |
| **Phase 4: Migration** | 5-7 days | Existing user migration, dual-running |
| **Phase 5: Cleanup** | 2-3 days | Remove legacy code, deprecate endpoints |
| **TOTAL** | **~3 weeks** | Full migration |
