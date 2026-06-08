# Implementation Overview

End-to-end walkthrough: create a customer, find their Global Account, register a passkey, fund the account, and execute a signed withdrawal.

## Prerequisites

- Platform configured with `USDB` in supported currencies
- Sandbox or production API credentials
- Access to Embedded Wallet Auth and Internal Accounts endpoints

## Setup

```bash
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
export GRID_CLIENT_ID="YOUR_SANDBOX_CLIENT_ID"
export GRID_CLIENT_SECRET="YOUR_SANDBOX_CLIENT_SECRET"
```

## Step 1: Create a Customer

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "ind-9f84e0c2",
    "region": "US",
    "email": "jane@example.com",
    "fullName": "Jane Doe",
    "birthDate": "1990-01-15",
    "nationality": "US"
  }'
```

A Global Account is provisioned automatically whenever a customer is created on a platform that has USDB in its supported currencies.

**Response:** `201 Created` with the new Customer ID. In sandbox, customer is KYC-approved immediately.

## Step 2: Find the Global Account

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts?customerId=Customer:...&currency=USDB" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Look for the account with `type: "EMBEDDED_WALLET"`:

```json
{
  "data": [
    {
      "id": "InternalAccount:ga-abc123",
      "type": "EMBEDDED_WALLET",
      "currency": "USDB",
      "balance": "0.00",
      "status": "ACTIVE"
    }
  ]
}
```

## Step 3: Register Authentication (Passkey)

### 3a. Start Passkey Registration

```bash
curl -X POST "$GRID_BASE_URL/auth/credentials" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "internalAccountId": "InternalAccount:ga-abc123",
    "credentialType": "PASSKEY",
    "name": "Jane\'s iPhone"
  }'
```

### 3b. Complete WebAuthn Ceremony

Use the WebAuthn API in the browser to complete registration:

```javascript
// Browser code
const credential = await navigator.credentials.create({
  publicKey: {
    challenge: Uint8Array.from(atob(serverResponse.challenge), c => c.charCodeAt(0)),
    rp: { name: "Your App", id: "your-domain.com" },
    user: {
      id: Uint8Array.from(atob(serverResponse.userId), c => c.charCodeAt(0)),
      name: "jane@example.com",
      displayName: "Jane Doe"
    },
    pubKeyCredParams: [{ alg: -7, type: "public-key" }],
    authenticatorSelection: { authenticatorAttachment: "platform" },
    timeout: 60000
  }
});

// Send credential back to server
const verificationResponse = {
  id: credential.id,
  rawId: btoa(String.fromCharCode(...new Uint8Array(credential.rawId))),
  type: credential.type,
  response: {
    clientDataJSON: btoa(String.fromCharCode(...new Uint8Array(credential.response.clientDataJSON))),
    attestationObject: btoa(String.fromCharCode(...new Uint8Array(credential.response.attestationObject)))
  }
};
```

## Step 4: Fund the Account

### Sandbox: Use Fund Endpoint

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:ga-abc123/fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "1000.00",
    "currency": "USDB"
  }'
```

### Production: Create Quote

Create a quote with the Global Account as destination:

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:platform-usd"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "InternalAccount:ga-abc123"
    },
    "amount": "1000.00",
    "amountCurrency": "USD"
  }'
```

Then execute the quote normally (no signature needed for incoming funds).

## Step 5: Add an External Bank Account

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "ACH",
    "accountNumber": "1234567890",
    "routingNumber": "021000021",
    "currency": "USD",
    "accountOwnerName": "Jane Doe"
  }'
```

## Step 6: Create a Withdrawal Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:ga-abc123"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "ExternalAccount:ach-xyz789"
    },
    "amount": "100.00",
    "amountCurrency": "USDB"
  }'
```

Response includes `payloadToSign`:

```json
{
  "id": "Quote:withdraw-123",
  "paymentInstructions": {
    "payloadToSign": "base64_encoded_payload..."
  }
}
```

## Step 7: Authenticate and Sign

### Get Session

```bash
curl -X POST "$GRID_BASE_URL/auth/credentials/{credentialId}/verify" \
  -H "Content-Type: application/json" \
  -d '{
    "challengeResponse": "..."
  }'
```

### Sign Payload

Use the session signing key to sign the payload:

```javascript
// Client-side signing
const signature = await crypto.subtle.sign(
  { name: "ECDSA", hash: "SHA-256" },
  sessionSigningKey,
  Uint8Array.from(atob(payloadToSign), c => c.charCodeAt(0))
);

const base64Signature = btoa(String.fromCharCode(...new Uint8Array(signature)));
```

## Step 8: Execute the Quote with Signature

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:withdraw-123/execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -H "Grid-Wallet-Signature: base64_signature_here"
```

## Complete!

The withdrawal is now processing. You'll receive webhook notifications for status updates.

## Where to Next

- [Authentication](./03-authentication.md) - More auth options
- [Client Keys & Signing](./04-client-keys-and-signing.md) - Detailed signing guide
- [Managing Sessions](./05-managing-sessions.md) - Session lifecycle
- [Exporting Wallet](./06-exporting-wallet.md) - Let users export seeds
