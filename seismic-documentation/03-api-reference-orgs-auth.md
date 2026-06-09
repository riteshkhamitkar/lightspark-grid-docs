# Seismic Orchestration — API Reference: Orgs & Auth

> **Full Documentation Analysis** | **Source:** https://orchestration.seismic.systems/docs/api/orgs, /docs/api/auth  
> **Analyzed:** June 9, 2026

---

## PART A: ORGS API

### Base Path: `/orgs/*`

An **org** is the tenant. Customers, API keys, KYB submissions, quotes, and orders all hang off an org.

---

### A.1 Org Endpoints

#### `GET /orgs` — List orgs
- Returns orgs the caller belongs to
- **Dashboard sessions only** (API key returns the org the key was minted in)

#### `POST /orgs` — Create a new org
- Caller becomes **owner**
- Request: `{ display_name: "Acme Inc." }`

#### `GET /org` — Read current org
- Returns the org bound to the current credential
- No org id in path — determined by auth context

#### `PATCH /org` — Update org
- Update display name and settings
- Request: `{ display_name: "New Name" }`

---

### A.2 Member Endpoints

#### `GET /org/members` — List members
#### `POST /org/members` — Invite a member
#### `PATCH /org/members/{user_id}` — Update member role
#### `DELETE /org/members/{user_id}` — Remove member
#### `DELETE /org/invitations/{invitation_id}` — Revoke pending invitation

---

### A.3 KYB Endpoints

The unified KYB profile + state for the org.

#### `GET /org/kyb` — Read KYB state
#### `PUT /org/kyb/profile` — Upsert KYB profile
- Accepts partial drafts
- Mirrors the dashboard KYB form sections

#### `POST /org/kyb/submit` — Submit KYB for review
- Org enters `submitted` state

#### `GET /org/kyb/state` — Read current verification state

### KYB Status Values
| Status | Meaning |
|--------|---------|
| `draft` | Profile saved but not submitted |
| `submitted` | Under review |
| `approved` | KYB approved — quoting and end-user onboarding unlocked |
| `rejected` | KYB rejected — see rejection reason |

---

### A.4 Webhook Management

#### `POST /org/webhooks/portal` — Get Svix portal URL
- Returns a signed URL to the Svix-hosted webhook console
- Configure endpoints, view delivery history, rotate signing secrets

---

### A.5 Org Types

#### Org
```typescript
{
  id: string;           // UUID
  display_name: string;
  status: "active" | "suspended";
  created_at: string;
  updated_at: string;
}
```

#### OrgMembership
```typescript
{
  user_id: string;
  org_id: string;
  role: OrgRole;
  joined_at: string;
}
```

#### OrgRole
```typescript
"owner" | "admin" | "member"
```

#### KybStatus
```typescript
"draft" | "submitted" | "approved" | "rejected"
```

---

## PART B: AUTH API

### Base Path: `/auth/*`

All endpoints use JSON request/response bodies.

---

### B.1 Auth Endpoints

#### `POST /auth/signup` — Create new account
```bash
POST /auth/signup
{
  "email": "alice@example.com",
  "password": "secure-password",
  "display_name": "Alice Liddell"
}

Response: TokenPair | LoginResponse
```

#### `POST /auth/login` — Exchange credentials for tokens
```bash
POST /auth/login
{
  "email": "alice@example.com",
  "password": "secure-password"
}

Response (no MFA): { access_token, refresh_token }
Response (MFA required): { status: "mfa_required", mfa_challenge_id }
```

#### `POST /auth/mfa/verify` — Redeem MFA challenge
```bash
POST /auth/mfa/verify
{
  "mfa_challenge_id": "mfc_...",
  "code": "123456"
}

Response: { access_token, refresh_token }
```

#### `POST /auth/mfa/enroll` — Start TOTP enrollment
```bash
POST /auth/mfa/enroll

Response: { qr_code_uri, secret }
```

#### `POST /auth/mfa/activate` — Confirm TOTP enrollment
```bash
POST /auth/mfa/activate
{
  "code": "123456"  // First code from authenticator
}
```

#### `POST /auth/refresh` — Rotate refresh token
```bash
POST /auth/refresh
{
  "refresh_token": "rt_..."
}

Response: { access_token, refresh_token }  // New pair, old refresh invalidated
```

#### `POST /auth/logout` — Revoke refresh token
```bash
POST /auth/logout
{
  "refresh_token": "rt_..."
}
```

#### `GET /auth/me` — Whoami
```bash
GET /auth/me
Authorization: Bearer <access_token>

Response: {
  id, email, display_name,
  memberships: [{ org_id, role }],
  mfa: { enrolled: boolean, active: boolean }
}
```

---

### B.2 API Key Endpoints

#### `POST /auth/api-keys` — Mint a new key
```bash
POST /auth/api-keys
Authorization: Bearer <access_token>
{ "label": "production-backend" }

Response: { "id": "ak_...", "secret": "seismic_sandbox_...", "label": "..." }
```
**Returns secret ONCE only.** Store it immediately.

#### `GET /auth/api-keys` — List keys
```bash
GET /auth/api-keys
Authorization: Bearer <access_token> | Api-Key: <key>

Response: [{ id, label, created_at, last_used_at, revoked_at }]
```

#### `DELETE /auth/api-keys/{id}` — Revoke a key
```bash
DELETE /auth/api-keys/ak_...
```

---

### B.3 Auth Types

#### TokenPair
```typescript
{
  access_token: string;   // JWT, minutes-long
  refresh_token: string;  // Opaque, days-long
}
```

#### LoginResponse
```typescript
// MFA required
{
  status: "mfa_required";
  mfa_challenge_id: string;
}

// Authenticated
{
  status: "authenticated";
  access_token: string;
  refresh_token: string;
}
```

---

## C. Enso Integration Notes

### What Enso Needs to Implement

| Feature | Endpoint | Priority |
|---------|----------|----------|
| KYB submission | `PUT /org/kyb/profile` + `POST /org/kyb/submit` | **HIGH** |
| API key management | `POST /auth/api-keys` | **HIGH** |
| Webhook configuration | `POST /org/webhooks/portal` | **HIGH** |
| Read KYB status | `GET /org/kyb/state` | **MEDIUM** |
| Member management | `GET/POST /org/members` | **LOW** |

### What Enso Does NOT Need

| Feature | Reason |
|---------|--------|
| Signup/Login/MFA | Using API keys for server-to-server |
| Refresh token flow | API keys don't expire |
| `/orgs` listing | Single org per API key |
| `GET /auth/me` | Only needed for dashboard sessions |

### Authentication for Enso's Backend

```typescript
import { SeismicOrchestrationClient } from "seismic-orchestration";

// Single initialization
const seismicApi = SeismicOrchestrationClient.sandbox({
  apiKey: process.env.SEISMIC_API_KEY,
});

// Use for all subsequent calls
const customers = await seismicApi.customers.list();
```

### Org Scoping
- All API calls are automatically scoped to the org that the API key belongs to
- No need to pass org_id in any request
- The key IS the org scope
