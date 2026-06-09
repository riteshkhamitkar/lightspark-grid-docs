# Seismic Orchestration — Authentication

> **Full Documentation Analysis** | **Source:** https://orchestration.seismic.systems/docs/authentication  
> **Analyzed:** June 9, 2026

---

## 1. Authentication Overview

Two auth schemes, **mutually exclusive** on a single request:

| Scheme | Header | Use Case |
|--------|--------|----------|
| **API Key** | `Api-Key: seismic_prod_...` | Server-to-server. Org-scoped (one key, one org). No login or refresh. |
| **Access Token** | `Authorization: Bearer <access-token>` | Browsers logged into dashboard.seismic.systems. Issued by email/password/MFA flow. |

**Sending both → `400 BothCredsProvided`**

The SDK handles either via separate constructor options (`apiKey` vs `tokenStore`); see `09-sdk-documentation.md`.

---

## 2. API Key Authentication (Recommended for Server-to-Server)

### How it works
- Mint an API key in the dashboard (or via `POST /auth/api-keys`)
- Key is **org-scoped**: one key belongs to exactly one org
- Key is **permanent** — doesn't expire (unless revoked)
- Include `Api-Key: <key>` header on every request
- No refresh dance needed

### Minting an API Key
**Dashboard:**
1. Settings → API keys → New key
2. Label it (e.g., `production-backend`)
3. Copy the secret — **shown once only**

**Programmatic:**
```bash
POST /auth/api-keys
Authorization: Bearer <access-token>  # Dashboard session required

Response: { "id": "ak_...", "secret": "seismic_sandbox_...", "label": "..." }
```

### Using API Keys
```bash
curl $SEISMIC_ORCHESTRATION_URL/v0/customers \
  -H "Api-Key: seismic_sandbox_..."
```

### API Key Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/auth/api-keys` | Mint a new key (returns secret once) |
| `GET` | `/auth/api-keys` | List keys for the current org |
| `DELETE` | `/auth/api-keys/{id}` | Revoke a key |

---

## 3. Access Token Authentication (Dashboard Sessions)

The dashboard authenticates with `Authorization: Bearer <access_token>`.

### Token Lifecycle
```
signup ─┐
        │
        ├─► login ──► tokens  ──────────────────► (use access until it 401s)
        │     │                                            │
        │     └──► mfa_required ──► mfaVerify ──► tokens   │
        │                                                  ▼
        │                                               refresh ── tokens
        │                                                  │
        └────────────────────────────────────────────── logout
```

### Token Types
| Token | Duration | Description |
|-------|----------|-------------|
| **Access token** | Minutes | Sent on every request. Stateless JWT verified against `JWT_ACCESS_SECRET`. |
| **Refresh token** | Days | Opaque. Stored server-side in `refresh_tokens`, one row per session. Rotated on every `refresh` call. |
| **MFA challenge id** | Seconds | Issued by `login` when MFA is active; redeemed by `mfaVerify` for a real token pair. |

### Single-Flight Refresh
If 50 concurrent requests 401 at the same time, the client refreshes **once**, not 50 times. The SDK implements this automatically.

---

## 4. Auth Endpoints (For Dashboard Sessions)

### Signup
```bash
POST /auth/signup
{
  "email": "alice@example.com",
  "password": "secure-password",
  "display_name": "Alice Liddell"
}
```
- Creates user + org membership
- Returns access token + refresh token (if no MFA required)

### Login (Password-Only)
```bash
POST /auth/login
{
  "email": "alice@example.com",
  "password": "secure-password"
}
```

**Response shapes:**
- MFA not enrolled → `TokenPair` (access + refresh tokens)
- MFA enrolled → `LoginResponse` with `status: "mfa_required"` + `mfa_challenge_id`

### MFA Verification
```bash
POST /auth/mfa/verify
{
  "mfa_challenge_id": "mfc_...",
  "code": "123456"
}
```
- Redeem MFA challenge for real tokens
- Returns `TokenPair`

### MFA Enrollment
```bash
POST /auth/mfa/enroll      # Start TOTP enrollment → returns QR code URI
POST /auth/mfa/activate    # Confirm enrollment with first code
```

### Refresh
```bash
POST /auth/refresh
{
  "refresh_token": "rt_..."
}
```
- Returns new `TokenPair`
- Previous refresh token is **invalidated** (rotation)

### Logout
```bash
POST /auth/logout
{
  "refresh_token": "rt_..."
}
```
- Revokes refresh token server-side

### Whoami
```bash
GET /auth/me
Authorization: Bearer <access-token>
```
- Returns full profile, memberships, MFA state

---

## 5. Auth Types

### TokenPair
```typescript
{
  access_token: string;   // Short-lived JWT
  refresh_token: string;  // Long-lived opaque token
}
```

### LoginResponse
```typescript
{
  status: "mfa_required";
  mfa_challenge_id: string;  // Seconds-long, pending MFA verification
}
// OR
{
  status: "authenticated";
  ...TokenPair;
}
```

---

## 6. Error Codes (Auth-Specific)

| Code | Status | Meaning |
|------|--------|---------|
| `PasswordTooShort` | 400 | Password below minimum length |
| `EmailAlreadyExists` | 409 | Signup email already registered |
| `InvalidCredentials` | 401 | Bad email/password on login |
| `MfaChallengeInvalid` | 401 | Wrong code, expired challenge, or clock skew > 30s |
| `RefreshTokenInvalid` | 401 | Refresh token expired or revoked. Send user to login |
| `BothCredsProvided` | 400 | Request had both `Api-Key:` and `Authorization: Bearer` headers |
| `ApiKeyNotFound` | 404 | Unknown key id (or belongs to different org) |
| `ApiKeyRevoked` | 401 | Key was valid but is now revoked |
| `AlreadyRevoked` | 409 | Revoke called on a key that was already revoked |

---

## 7. Enso Backend Integration Recommendations

### For Server-to-Server (Recommended)
```typescript
// Enso backend - server-to-server
import { SeismicOrchestrationClient } from "seismic-orchestration";

const api = SeismicOrchestrationClient.sandbox({
  apiKey: process.env.SEISMIC_API_KEY,  // Org-scoped, permanent
});
```

### Environment Variables for Enso
```bash
# Seismic Integration
SEISMIC_API_KEY=seismic_sandbox_...        # Sandbox key
SEISMIC_API_KEY_PROD=seismic_prod_...      # Production key
SEISMIC_BASE_URL=https://orchestration-sandbox.seismictest.net/api
```

### No User Login Flow Needed
Since Enso's backend will use API keys, the auth endpoints (signup, login, MFA, refresh) are **not needed** for production traffic. They are only relevant if building a dashboard integration.

---

## 8. Auth + Scoping for Orgs

### URL Conventions
| URL | Meaning |
|-----|---------|
| `/orgs` (plural) | The caller's set of memberships. Listing + creation. |
| `/org` (singular) | The **current** org that the credential is scoped to. |

### With API Key
- The org the key was minted in
- Permanent — keys don't switch context

### With Dashboard Session
- Bearer token represents a **user**, not an org
- JWT claim is `sub: <user_id>`, with no embedded org id
- User can belong to multiple orgs; "Switch org" UI determines active org

---

## 9. Picking an Org (Dashboard Sessions Only)

A bearer token represents a **user**, not an org. The JWT claim is `sub: <user_id>`, with no embedded org id. For dashboard sessions, the active org is determined by the session context.

---

## 10. Comparison: Enso's Current Auth vs Seismic

| Aspect | Enso Current (KYC Module) | Seismic |
|--------|--------------------------|---------|
| Auth method | JWT Bearer token | API Key or Bearer token |
| Token scope | Per-user | Per-org (API Key) |
| Token refresh | Manual | Automatic (SDK handles 401 refresh) |
| MFA | Not used | Required for dashboard sessions |
| API Key rotation | N/A | Via dashboard or API |

---

## 11. Migration Notes

### For Enso Backend
1. **Mint an API key** from Seismic dashboard (one per environment)
2. **Store as environment variable** (`SEISMIC_API_KEY`)
3. **Use API Key auth** for all server-to-server calls
4. **No need to implement** login/signup/MFA/refresh flows
5. The SDK handles single-flight refresh automatically if using token auth
