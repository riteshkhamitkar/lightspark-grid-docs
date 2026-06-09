# Seismic Orchestration — Webhooks

> **Full Documentation Analysis** | **Source:** https://orchestration.seismic.systems/docs/webhooks  
> **Analyzed:** June 9, 2026

---

## 1. Webhooks Overview

Seismic delivers **signed HTTPS POST** requests to your configured endpoint(s) whenever:
- A customer's verification status changes
- An account is created or updated
- An order changes state

### Key Features
| Feature | Description |
|---------|-------------|
| **Delivery** | Signed HTTPS POST via Svix |
| **Payload** | Full resource snapshot inline (no follow-up GET needed) |
| **Console** | Svix-hosted console for configuration, history, retries |
| **Signing** | Svix-compatible signature verification |
| **Idempotency** | Event IDs for deduplication |

### Webhook Console Access
```bash
POST /org/webhooks/portal
Api-Key: seismic_...

Response: { "url": "https://eu.svix.com/portal/..." }
```
- Use this URL to give your team access to the webhook dashboard
- Configure endpoints, view delivery logs, rotate signing secrets

---

## 2. Event Catalog

Five event types, grouped by entity:

| Entity | Events | Page |
|--------|--------|------|
| **Customer (KYC/KYB)** | `customer.kyc.status_changed`, `customer.kyb.status_changed` | Customer events |
| **Account** | `account.created`, `account.updated` | Account events |
| **Order** | `order.status_changed` | Order events |

**Important:** Events only fire on **forward transitions** through the status ladder. Backwards or no-op transitions are **suppressed at the source**.

---

## 3. Envelope Format

Every payload is a JSON envelope:

```json
{
  "event_type": "customer.kyc.status_changed",
  "data": {
    "...": "see per-entity sections below"
  }
}
```

- `additionalProperties` is `false` on every variant
- Treat unknown top-level keys in `data` as a contract violation

---

## 4. Customer Events

### `customer.kyc.status_changed`
Fired when an individual customer's KYC verification status changes.

```json
{
  "event_type": "customer.kyc.status_changed",
  "data": {
    "customer_id": "2f10aef0-c1d1-4a3e-8b1d-1234567890ab",
    "status": "approved",
    "kyc_profile": {
      "identity": {
        "first_name": "Ada",
        "last_name": "Lovelace",
        "date_of_birth": "1990-04-12",
        "nationality": "US"
      },
      "addresses": {
        "residential": {
          "line1": "123 Main St",
          "city": "Brooklyn",
          "state": "NY",
          "postal_code": "11201",
          "country": "US"
        }
      },
      "contact": {
        "email": "ada@example.com",
        "phone": "+14155551234"
      },
      "employment": {
        "occupation": "Software Developer",
        "employment_status": "employed"
      },
      "risk": {
        "source_of_funds": "salary",
        "account_purpose": "personal_use"
      },
      "volumes": {
        "expected_monthly_volume": "1000_4999"
      },
      "jurisdictions": {
        "tax_id": "123-45-6789",
        "tax_id_country": "US"
      },
      "attestations": {
        "terms_accepted": true
      },
      "documents": {
        "...": "document references"
      }
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `customer_id` | UUID | The customer whose KYC status changed |
| `status` | KyStatus | New aggregate status |
| `kyc_profile` | KycProfile | Full saved profile at event time |

### `customer.kyb.status_changed`
Fired when a business customer's KYB verification status changes.

```json
{
  "event_type": "customer.kyb.status_changed",
  "data": {
    "customer_id": "8e3b6a3c-8e3b-4a3c-8e3b-6a3c8e3b6a3c",
    "status": "approved",
    "kyb_profile": {
      "identity": {
        "legal_name": "Acme Inc.",
        "entity_type": "corporation",
        "tax_id": "12-3456789"
      },
      "addresses": {
        "registered": { "...": "..." }
      },
      "contact": {
        "email": "legal@acme.com"
      },
      "operations": {
        "jurisdictions": ["US"],
        "source_of_funds": "business_revenue"
      },
      "associated_persons": [
        {
          "name": "John Doe",
          "role": "ubo",
          "ownership_percentage": 25
        }
      ]
    }
  }
}
```

### KyStatus Values
| Status | Meaning |
|--------|---------|
| `not_started` | No KYC/KYB initiated |
| `submitted` | Profile submitted, awaiting review |
| `under_review` | Actively being reviewed |
| `approved` | **KYC/KYB approved** — customer can transact |
| `rejected` | KYC/KYB rejected — see rejection reason |
| `unknown` | Status cannot be determined |

---

## 5. Account Events

### `account.created`
Fired when a new account is created for a customer.

```json
{
  "event_type": "account.created",
  "data": {
    "account_id": "acc_1a2b...",
    "customer_id": "2f10aef0-...",
    "account": {
      "id": "acc_1a2b...",
      "kind": "fiat",
      "shape": "us",
      "currency": "USD",
      "rails": ["ach", "wire"],
      "details": {
        "account_number": "...",
        "routing_number": "..."
      },
      "status": "active",
      "created_at": "2026-01-15T10:30:00Z"
    }
  }
}
```

### `account.updated`
Fired when an account's status or details change.

```json
{
  "event_type": "account.updated",
  "data": {
    "account_id": "acc_1a2b...",
    "customer_id": "2f10aef0-...",
    "account": {
      "...": "full account snapshot"
    }
  }
}
```

---

## 6. Order Events

### `order.status_changed`
Fired when an order moves forward in its lifecycle.

```json
{
  "event_type": "order.status_changed",
  "data": {
    "order_id": "9b32f5cd-7c0e-4d2a-9111-fedcba987654",
    "customer_id": "2f10aef0-c1d1-4a3e-8b1d-1234567890ab",
    "status": "completed",
    "order": {
      "id": "9b32f5cd-...",
      "status": "completed",
      "legs": [{ "...": "..." }],
      "source": { "...": "..." },
      "destination": { "...": "..." },
      "created_at": "2026-01-15T10:30:00Z",
      "updated_at": "2026-01-15T11:00:00Z"
    }
  }
}
```

### OrderStatus Values
| Status | Meaning |
|--------|---------|
| `awaiting_funds` | Order accepted, waiting for customer to send funds |
| `pending` | Funds received, processing initiated |
| `processing` | Actively being processed |
| `completed` | **Order completed successfully** |
| `expired` | Order expired (quote timed out) |
| `failed` | Order failed |
| `canceled` | Order was canceled |
| `unknown` | Status cannot be determined |

### Order Lifecycle
```
awaiting_funds → pending → processing → completed
     ↓
  canceled

Or:
awaiting_funds → pending → processing → failed
```

---

## 7. Delivery Semantics

### Signature Verification
- Uses Svix-compatible signing
- Libraries available: `svix` on npm, PyPI, crates.io
- Any Standard Webhooks library works out of the box

### Verification Code Example (Node.js)
```typescript
import { Webhook } from "svix";

const webhookSecret = process.env.SEISMIC_WEBHOOK_SECRET;

app.post("/webhooks/seismic", (req, res) => {
  const wh = new Webhook(webhookSecret);
  
  try {
    const payload = wh.verify(req.body, req.headers);
    
    switch (payload.event_type) {
      case "customer.kyc.status_changed":
        await handleKycStatusChanged(payload.data);
        break;
      case "order.status_changed":
        await handleOrderStatusChanged(payload.data);
        break;
      // ... etc
    }
    
    res.status(200).send("OK");
  } catch (err) {
    res.status(400).send("Invalid signature");
  }
});
```

### Retry Policy
- Svix handles automatic retries with exponential backoff
- Failed deliveries are retried multiple times
- View retry status in the Svix console

### Backfill Caveat
The embedded resource snapshot (`kyc_profile`, `kyb_profile`, `account`, `order`) is **optional** in the published JSON Schema:
- **Fresh events** → always include the snapshot
- **Historical replays** from Svix console → may encounter older rows without it
- **Always fall back to a REST read** if the snapshot is missing

---

## 8. Subscribing to Webhooks

### Step 1: Get Portal URL
```bash
POST /org/webhooks/portal
Api-Key: seismic_...

Response: { "url": "https://eu.svix.com/portal/..." }
```

### Step 2: Configure Endpoint
In the Svix console:
1. Add endpoint URL (e.g., `https://api.enso.com/webhooks/seismic`)
2. Select event types to subscribe to
3. Copy signing secret
4. Save configuration

### Step 3: Verify Signatures
Use the signing secret to verify webhook signatures in your handler.

---

## 9. Enso Integration: Webhook Handler

### Required Webhook Endpoint
```typescript
// POST /api/v1/webhooks/seismic
import { Webhook } from "svix";
import { prisma } from "../prisma";

const SEISMIC_WEBHOOK_SECRET = process.env.SEISMIC_WEBHOOK_SECRET;

export async function handleSeismicWebhook(req: Request, res: Response) {
  const wh = new Webhook(SEISMIC_WEBHOOK_SECRET);
  
  let payload;
  try {
    payload = wh.verify(req.body, req.headers);
  } catch (err) {
    return res.status(400).json({ error: "Invalid signature" });
  }
  
  const { event_type, data } = payload;
  
  switch (event_type) {
    case "customer.kyc.status_changed": {
      await handleKycStatusChanged(data);
      break;
    }
    case "account.created":
    case "account.updated": {
      await handleAccountEvent(data);
      break;
    }
    case "order.status_changed": {
      await handleOrderStatusChanged(data);
      break;
    }
    default: {
      console.warn(`Unknown webhook event: ${event_type}`);
    }
  }
  
  res.status(200).send("OK");
}

async function handleKycStatusChanged(data: any) {
  const { customer_id, status, kyc_profile } = data;
  
  // Map Seismic customer to Enso user
  const user = await prisma.user.findFirst({
    where: { seismicCustomerId: customer_id }
  });
  
  if (!user) {
    console.error(`No user found for Seismic customer: ${customer_id}`);
    return;
  }
  
  // Update KYC status
  await prisma.kycProfile.update({
    where: { userId: user.id },
    data: {
      kycStatus: status,
      // Sync profile data if provided
      firstName: kyc_profile?.identity?.first_name,
      lastName: kyc_profile?.identity?.last_name,
      // ... etc
    }
  });
  
  // If approved, trigger account creation
  if (status === "approved") {
    await createSeismicAccounts(user.id, customer_id);
  }
}

async function handleOrderStatusChanged(data: any) {
  const { order_id, status, order } = data;
  
  // Update order status in Enso DB
  await prisma.order.updateMany({
    where: { seismicOrderId: order_id },
    data: { status }
  });
  
  // Notify user if needed
  if (status === "completed") {
    await notifyUserOrderCompleted(order_id);
  }
}
```

### Environment Variables
```bash
SEISMIC_WEBHOOK_SECRET=whsec_...
```

---

## 10. Comparison: Enso's Current Webhooks vs Seismic

| Aspect | Enso Current | Seismic |
|--------|-------------|---------|
| **Sumsub webhook** | `POST /api/v1/kyc/webhook/sumsub` | Replaced by `customer.kyc.status_changed` |
| **Bridge webhook** | `POST /api/v1/bridge/webhook` | Replaced by `order.status_changed` + `account.*` |
| **Signature** | HMAC-SHA256 (custom) | Svix standard |
| **Payload** | Minimal (IDs only) | Full resource snapshot inline |
| **Retry** | Manual | Automatic (Svix) |
| **Console** | None | Svix-hosted portal |
| **Event types** | 2 sources (Sumsub + Bridge) | 5 unified events |

### Migration Strategy
1. **Replace Sumsub webhook** → Subscribe to `customer.kyc.status_changed`
2. **Replace Bridge webhook** → Subscribe to `order.status_changed` + `account.*`
3. **Unify webhook handler** → Single `/webhooks/seismic` endpoint
4. **Remove legacy handlers** → After migration complete
