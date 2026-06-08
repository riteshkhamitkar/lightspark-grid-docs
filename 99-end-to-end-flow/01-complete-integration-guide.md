# Complete End-to-End Integration Guide

> This guide walks through the entire Lightspark Grid integration from onboarding to production, with real-world examples for a US-to-India payout platform.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Step 1: Platform Setup](#step-1-platform-setup)
3. [Step 2: Customer Onboarding with SumSub KYC](#step-2-customer-onboarding-with-sumsub-kyc)
4. [Step 3: Account Management](#step-3-account-management)
5. [Step 4: US to India Payout Flow](#step-4-us-to-india-payout-flow)
6. [Step 5: Global Accounts (Self-Custody)](#step-5-global-accounts-self-custody)
7. [Step 6: Webhook Handling](#step-6-webhook-handling)
8. [Step 7: Error Handling & Retries](#step-7-error-handling--retries)
9. [Step 8: Sandbox Testing](#step-8-sandbox-testing)
10. [Step 9: Production Deployment](#step-9-production-deployment)
11. [Real-World Example: Complete US to India Transfer](#real-world-example-complete-us-to-india-transfer)
12. [SumSub Integration Deep Dive](#sumsub-integration-deep-dive)
13. [API Quick Reference](#api-quick-reference)
14. [Troubleshooting Guide](#troubleshooting-guide)

---

## Architecture Overview

### System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         YOUR PLATFORM                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │   Web    │  │  Mobile  │  │ Backend  │  │  Database    │   │
│  │   App    │  │   App    │  │  Server  │  │  (Postgres)  │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────────────┘   │
│       │             │             │                              │
│       └─────────────┴─────────────┘                              │
│                     │                                            │
│              ┌──────┴──────┐                                    │
│              │  Webhook    │                                    │
│              │  Handler    │                                    │
│              └──────┬──────┘                                    │
└─────────────────────┼───────────────────────────────────────────┘
                      │
              ┌───────┴────────┐
              │  LIGHTSPARK    │
              │     GRID       │
              │    (REST API)  │
              └───────┬────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
   ┌────┴────┐  ┌────┴────┐  ┌────┴────┐
   │  SumSub │  │  Banks  │  │Bitcoin/ │
   │  (KYC)  │  │ (Rails) │  │Lightning│
   └─────────┘  └─────────┘  └─────────┘
```

### Data Flow

```
1. Customer signs up on your platform
2. You create customer in Grid via API
3. SumSub handles KYC verification
4. Customer adds bank account (UPI for India)
5. Customer funds their USD internal account
6. Customer creates quote (USD -> INR)
7. Grid locks exchange rate
8. Customer executes quote
9. Grid converts USD -> INR via BTC/Lightning
10. Grid delivers INR via UPI to Indian bank
11. Webhook notifications at each step
```

---

## Step 1: Platform Setup

### 1.1 Create Grid Account

1. Go to https://app.lightspark.com
2. Sign up for a business account
3. Complete platform verification
4. Access the dashboard

### 1.2 Generate API Credentials

```bash
# Dashboard -> API Keys -> Generate New Key
# Copy Client ID and Client Secret
export GRID_CLIENT_ID="your_client_id"
export GRID_CLIENT_SECRET="your_client_secret"
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
```

### 1.3 Configure Platform

```bash
# Set supported currencies for US-to-India corridor
curl -X PATCH "$GRID_BASE_URL/platform" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "supportedCurrencies": ["USD", "INR", "BTC", "USDC"],
    "webhookUrl": "https://your-domain.com/webhooks/grid",
    "name": "Your Remittance Platform"
  }'
```

### 1.4 Set Up Webhook Endpoint

```javascript
// Node.js Express webhook handler
const express = require('express');
const crypto = require('crypto');
const app = express();

app.use(express.raw({ type: 'application/json' }));

const WEBHOOK_SECRET = process.env.GRID_WEBHOOK_SECRET;

function verifyWebhook(payload, signature, timestamp, secret) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expected, 'hex')
  );
}

app.post('/webhooks/grid', async (req, res) => {
  const signature = req.headers['x-grid-signature'];
  const timestamp = req.headers['x-grid-timestamp'];

  if (!verifyWebhook(req.body, signature, timestamp, WEBHOOK_SECRET)) {
    return res.status(401).send('Invalid signature');
  }

  const event = JSON.parse(req.body);

  // Process asynchronously
  processWebhookEvent(event);

  res.status(200).send('OK');
});

async function processWebhookEvent(event) {
  console.log(`Received event: ${event.eventType}`);

  switch (event.eventType) {
    case 'customer.status_change':
      await handleCustomerStatusChange(event.data);
      break;
    case 'verification.status_change':
      await handleVerificationStatusChange(event.data);
      break;
    case 'transaction.status_change':
      await handleTransactionStatusChange(event.data);
      break;
    case 'incoming_payment.received':
      await handleIncomingPayment(event.data);
      break;
  }
}

app.listen(3000, () => console.log('Webhook server running'));
```

---

## Step 2: Customer Onboarding with SumSub KYC

### 2.1 Create Customer

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "user_123456",
    "region": "US",
    "fullName": "Raj Patel",
    "birthDate": "1985-03-15",
    "nationality": "US",
    "email": "raj.patel@email.com",
    "phone": "+1-555-123-4567",
    "address": {
      "line1": "789 Oak Avenue",
      "line2": "Apt 12B",
      "city": "San Francisco",
      "state": "CA",
      "postalCode": "94102",
      "country": "US"
    },
    "identification": {
      "type": "SSN",
      "number": "123-45-6789"
    }
  }'
```

Response:
```json
{
  "id": "Customer:0195a1b2-c3d4-5e6f-7a8b-9c0d1e2f3a4b",
  "platformCustomerId": "user_123456",
  "customerType": "INDIVIDUAL",
  "kycStatus": "NOT_STARTED",
  "fullName": "Raj Patel",
  "email": "raj.patel@email.com",
  "createdAt": "2025-01-15T10:00:00Z"
}
```

### 2.2 Generate SumSub KYC Link

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:0195a1b2-c3d4-5e6f-7a8b-9c0d1e2f3a4b/kyc-link" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "redirectUrl": "https://your-app.com/kyc-complete?userId=user_123456"
  }'
```

Response:
```json
{
  "url": "https://verify.sumsub.com/idensic/liveness/#/uni?_si=abc123...",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "expiresAt": "2025-01-16T10:00:00Z"
}
```

### 2.3 Redirect Customer to KYC

```javascript
// Frontend - redirect user to SumSub hosted KYC
window.location.href = kycLink.url;
```

### 2.4 Customer Completes KYC (SumSub Flow)

The SumSub flow includes:
1. **Document Upload** - Passport or Driver's License
2. **Liveness Check** - Selfie verification
3. **Address Verification** - Proof of address (if needed)
4. **Review** - Automated + manual review

### 2.5 Handle KYC Completion Webhook

```json
{
  "eventType": "verification.status_change",
  "timestamp": "2025-01-15T11:30:00Z",
  "data": {
    "verificationId": "Verification:0195...",
    "customerId": "Customer:0195a1b2-c3d4-5e6f-7a8b-9c0d1e2f3a4b",
    "status": "APPROVED",
    "previousStatus": "IN_REVIEW",
    "kycStatus": "APPROVED",
    "errors": [],
    "completedAt": "2025-01-15T11:30:00Z"
  }
}
```

```javascript
async function handleVerificationStatusChange(data) {
  if (data.status === 'APPROVED') {
    // Enable payment features for customer
    await db.users.update(data.customerId, {
      kycStatus: 'APPROVED',
      paymentsEnabled: true,
      kycCompletedAt: new Date()
    });

    // Send welcome email
    await sendEmail(data.customerId, 'KYC Approved!', 
      'You can now start sending money to India.');

    // Create notification in app
    await createNotification(data.customerId, 'Your identity verification is complete!');
  } else if (data.status === 'REJECTED') {
    await db.users.update(data.customerId, {
      kycStatus: 'REJECTED',
      paymentsEnabled: false
    });

    await sendEmail(data.customerId, 'Verification Failed',
      'Please contact support for assistance.');
  } else if (data.status === 'RESOLVE_ERRORS') {
    // Show customer what's missing
    const errors = data.errors.map(e => e.reason).join(', ');
    await createNotification(data.customerId, 
      `Additional information needed: ${errors}`);
  }
}
```

---

## Step 3: Account Management

### 3.1 Get Customer's Internal Accounts

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts?customerId=Customer:0195a1b2-c3d4-5e6f-7a8b-9c0d1e2f3a4b" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Response:
```json
{
  "data": [
    {
      "id": "InternalAccount:usd-abc123",
      "type": "CHECKING",
      "currency": "USD",
      "balance": "0.00",
      "status": "ACTIVE",
      "paymentInstructions": {
        "accountNumber": "1234567890",
        "routingNumber": "021000021",
        "bankName": "Lightspark Bank",
        "reference": "CUST_0195A1B2"
      }
    },
    {
      "id": "InternalAccount:inr-def456",
      "type": "CHECKING",
      "currency": "INR",
      "balance": "0.00",
      "status": "ACTIVE"
    }
  ]
}
```

### 3.2 Show Funding Instructions to Customer

```javascript
// Display to customer how to fund their USD account
const fundingInstructions = {
  bankName: 'Lightspark Bank',
  accountNumber: '1234567890',
  routingNumber: '021000021',
  reference: 'CUST_0195A1B2'
};

// Customer sends ACH push from their bank
```

### 3.3 Handle Deposit Webhook

```json
{
  "eventType": "incoming_payment.received",
  "timestamp": "2025-01-15T14:00:00Z",
  "data": {
    "accountId": "InternalAccount:usd-abc123",
    "customerId": "Customer:0195a1b2-c3d4-5e6f-7a8b-9c0d1e2f3a4b",
    "amount": "5000.00",
    "currency": "USD",
    "reference": "CUST_0195A1B2"
  }
}
```

```javascript
async function handleIncomingPayment(data) {
  // Update balance in your database
  await db.accounts.update(data.accountId, {
    balance: data.amount,
    currency: data.currency
  });

  // Notify customer
  await createNotification(data.customerId, 
    `Received $${data.amount} USD. Ready to send to India!`);
}
```

### 3.4 Add Beneficiary (India UPI Account)

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:0195a1b2-c3d4-5e6f-7a8b-9c0d1e2f3a4b/external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "UPI",
    "upiId": "family@okicici",
    "currency": "INR",
    "accountOwnerName": "Family Member"
  }'
```

Response:
```json
{
  "id": "ExternalAccount:upi-789ghi",
  "accountType": "UPI",
  "upiId": "family@okicici",
  "currency": "INR",
  "status": "ACTIVE"
}
```

---

## Step 4: US to India Payout Flow

### 4.1 Customer Initiates Transfer

Customer wants to send $500 to family in India.

### 4.2 Create Quote (USD -> INR)

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "sourceType": "ACCOUNT",
      "accountId": "InternalAccount:usd-abc123"
    },
    "destination": {
      "destinationType": "ACCOUNT",
      "accountId": "ExternalAccount:upi-789ghi"
    },
    "amount": "500.00",
    "amountCurrency": "USD"
  }'
```

Response:
```json
{
  "id": "Quote:quote-xyz789",
  "sourceAmount": "500.00",
  "sourceAmountCurrency": "USD",
  "destinationAmount": "41500.00",
  "destinationAmountCurrency": "INR",
  "exchangeRate": "83.00",
  "fees": "7.50",
  "feesCurrency": "USD",
  "totalCost": "507.50",
  "totalCostCurrency": "USD",
  "expiresAt": "2025-01-15T15:05:00Z"
}
```

### 4.3 Show Quote to Customer

```javascript
function displayQuote(quote) {
  return {
    sending: `$${quote.sourceAmount} ${quote.sourceAmountCurrency}`,
    receiving: `Rs. ${quote.destinationAmount} ${quote.destinationAmountCurrency}`,
    exchangeRate: `1 USD = ${quote.exchangeRate} INR`,
    fee: `$${quote.fees}`,
    total: `$${quote.totalCost}`,
    expiresIn: '5 minutes'
  };
}

// Show confirmation dialog
// "You send: $500 USD"
// "They receive: Rs. 41,500 INR"
// "Exchange rate: 1 USD = 83.00 INR"
// "Fee: $7.50"
// "Total cost: $507.50"
```

### 4.4 Execute Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:quote-xyz789/execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: transfer-user123456-20250115-001"
```

Response:
```json
{
  "id": "Transaction:tx-abc123",
  "status": "PENDING",
  "type": "TRANSFER_OUT",
  "amount": "500.00",
  "currency": "USD",
  "destinationAmount": "41500.00",
  "destinationCurrency": "INR",
  "quoteId": "Quote:quote-xyz789",
  "createdAt": "2025-01-15T15:00:00Z"
}
```

### 4.5 Track Transaction via Webhooks

```json
// Transaction created
{
  "eventType": "transaction.status_change",
  "data": {
    "transactionId": "Transaction:tx-abc123",
    "status": "PENDING",
    "previousStatus": null
  }
}

// Transaction in flight (being processed)
{
  "eventType": "transaction.status_change",
  "data": {
    "transactionId": "Transaction:tx-abc123",
    "status": "IN_FLIGHT",
    "previousStatus": "PENDING"
  }
}

// Transaction completed
{
  "eventType": "transaction.status_change",
  "data": {
    "transactionId": "Transaction:tx-abc123",
    "status": "COMPLETED",
    "previousStatus": "IN_FLIGHT",
    "completedAt": "2025-01-15T15:02:00Z"
  }
}
```

### 4.6 Notify Customer of Completion

```javascript
async function handleTransactionStatusChange(data) {
  if (data.status === 'COMPLETED') {
    await createNotification(data.customerId,
      `Rs. 41,500 sent successfully to family@okicici!`);

    await sendEmail(data.customerId, 'Transfer Complete',
      `Your transfer of $500 USD to India has been completed. ` +
      `Rs. 41,500 has been delivered via UPI to family@okicici.`);
  } else if (data.status === 'FAILED') {
    await createNotification(data.customerId,
      'Transfer failed. Please try again or contact support.');
  }
}
```

---

## Step 5: Global Accounts (Self-Custody)

### 5.1 Create Customer with Global Account

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "global-user-001",
    "region": "US",
    "currencies": ["USD", "USDB"],
    "fullName": "Jane Doe",
    "birthDate": "1990-01-15",
    "nationality": "US",
    "email": "jane@example.com"
  }'
```

A Global Account (type: EMBEDDED_WALLET, currency: USDB) is auto-created.

### 5.2 Register Passkey

```bash
# 1. Create credential
curl -X POST "$GRID_BASE_URL/auth/credentials" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "internalAccountId": "InternalAccount:ga-abc123",
    "credentialType": "PASSKEY",
    "name": "Jane\'s iPhone"
  }'

# 2. Complete WebAuthn ceremony in browser
# 3. Credential registered
```

### 5.3 Fund Global Account

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:ga-abc123/fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"amount": "1000.00", "currency": "USDB"}'
```

### 5.4 Add Bank Account for Withdrawal

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:global-user-001/external-accounts" \
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

### 5.5 Create Withdrawal Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {"sourceType": "ACCOUNT", "accountId": "InternalAccount:ga-abc123"},
    "destination": {"destinationType": "ACCOUNT", "accountId": "ExternalAccount:ach-789"},
    "amount": "100.00",
    "amountCurrency": "USDB"
  }'
```

Response includes `payloadToSign`:
```json
{
  "paymentInstructions": {
    "payloadToSign": "base64_encoded_payload..."
  }
}
```

### 5.6 Sign and Execute

```javascript
// Client-side signing
const signature = await crypto.subtle.sign(
  { name: "ECDSA", hash: "SHA-256" },
  sessionSigningKey,
  base64ToBuffer(payloadToSign)
);
const base64Signature = btoa(String.fromCharCode(...new Uint8Array(signature)));

// Execute with signature
await fetch(`${GRID_BASE_URL}/quotes/${quoteId}/execute`, {
  method: 'POST',
  headers: {
    'Authorization': `Basic ${btoa(`${CLIENT_ID}:${CLIENT_SECRET}`)}`,
    'Content-Type': 'application/json',
    'Grid-Wallet-Signature': base64Signature
  }
});
```

---

## Step 6: Webhook Handling

### Complete Webhook Handler

```javascript
const express = require('express');
const crypto = require('crypto');
const app = express();

app.use(express.raw({ type: 'application/json' }));

const WEBHOOK_SECRET = process.env.GRID_WEBHOOK_SECRET;

function verifyWebhook(payload, signature, timestamp, secret) {
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - timestamp) > 300) return false;

  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expected, 'hex')
  );
}

app.post('/webhooks/grid', async (req, res) => {
  const signature = req.headers['x-grid-signature'];
  const timestamp = req.headers['x-grid-timestamp'];

  if (!verifyWebhook(req.body, signature, timestamp, WEBHOOK_SECRET)) {
    return res.status(401).send('Invalid signature');
  }

  const event = JSON.parse(req.body);

  // Process in background
  processWebhookEvent(event).catch(console.error);

  res.status(200).send('OK');
});

async function processWebhookEvent(event) {
  console.log(`[${new Date().toISOString()}] Event: ${event.eventType}`);

  switch (event.eventType) {
    case 'customer.status_change':
      await handleCustomerStatusChange(event.data);
      break;
    case 'verification.status_change':
      await handleKycStatusChange(event.data);
      break;
    case 'transaction.status_change':
      await handleTransactionUpdate(event.data);
      break;
    case 'incoming_payment.received':
      await handleDeposit(event.data);
      break;
    case 'internal_account.status_change':
      await handleBalanceUpdate(event.data);
      break;
    case 'card.state_change':
      await handleCardStateChange(event.data);
      break;
    default:
      console.log(`Unhandled event: ${event.eventType}`);
  }
}

async function handleCustomerStatusChange(data) {
  await db.users.updateByGridId(data.customerId, {
    status: data.newStatus,
    updatedAt: new Date()
  });
}

async function handleKycStatusChange(data) {
  if (data.status === 'APPROVED') {
    await db.users.updateByGridId(data.customerId, {
      kycStatus: 'APPROVED',
      paymentsEnabled: true
    });
    await notifyUser(data.customerId, 'Identity verified! Start sending money.');
  }
}

async function handleTransactionUpdate(data) {
  await db.transactions.update(data.transactionId, {
    status: data.status,
    updatedAt: new Date()
  });

  if (data.status === 'COMPLETED') {
    await notifyUser(data.customerId, 
      `Transfer completed! ${data.destinationAmount} ${data.destinationCurrency} delivered.`);
  } else if (data.status === 'FAILED') {
    await notifyUser(data.customerId, 
      'Transfer failed. Please try again or contact support.');
  }
}

async function handleDeposit(data) {
  await db.accounts.updateBalance(data.accountId, data.balance);
  await notifyUser(data.customerId, 
    `Received $${data.amount} ${data.currency}`);
}

async function handleBalanceUpdate(data) {
  await db.accounts.updateBalance(data.accountId, data.balance);
}

async function handleCardStateChange(data) {
  await db.cards.update(data.cardId, { status: data.newState });
}

app.listen(3000);
```

---

## Step 7: Error Handling & Retries

### Error Handler

```javascript
class GridErrorHandler {
  static async handle(error, context) {
    console.error(`Grid API Error [${context}]:`, error);

    switch (error.code) {
      case 'QUOTE_EXPIRED':
        // Create new quote
        return { action: 'RETRY_WITH_NEW_QUOTE' };

      case 'INSUFFICIENT_FUNDS':
        // Notify user to add funds
        await notifyUser(context.customerId, 'Insufficient funds. Please add money to your account.');
        return { action: 'NOTIFY_USER' };

      case 'KYC_NOT_APPROVED':
        // Redirect to KYC
        await notifyUser(context.customerId, 'Please complete identity verification.');
        return { action: 'REDIRECT_KYC' };

      case 'BANK_REJECTED':
        // Check account details
        await notifyUser(context.customerId, 'Bank rejected the transfer. Please verify account details.');
        return { action: 'VERIFY_ACCOUNT' };

      case 'RATE_LIMIT_EXCEEDED':
        // Wait and retry
        await sleep(60000);
        return { action: 'RETRY' };

      default:
        // Log and alert
        await alertOps(`Unhandled Grid error: ${error.code}`);
        return { action: 'ALERT_OPS' };
    }
  }
}

// Retry with exponential backoff
async function retryWithBackoff(fn, maxRetries = 5) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      if (!isRetryable(error)) throw error;

      const delay = Math.pow(2, i) * 1000;
      await sleep(delay);
    }
  }
}

function isRetryable(error) {
  return [500, 502, 503, 429].includes(error.statusCode);
}
```

---

## Step 8: Sandbox Testing

### Test Scenarios

```bash
#!/bin/bash
# test-us-to-india.sh

GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
GRID_CLIENT_ID="your_sandbox_id"
GRID_CLIENT_SECRET="your_sandbox_secret"

# 1. Create customer (KYC auto-approved)
echo "Creating customer..."
CUSTOMER=$(curl -s -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "test-001",
    "fullName": "Test User",
    "lastName": "TestUser",
    "region": "US",
    "email": "test@example.com"
  }')
CUSTOMER_ID=$(echo $CUSTOMER | jq -r '.id')
echo "Customer: $CUSTOMER_ID"

# 2. Fund account
echo "Funding account..."
ACCOUNT=$(curl -s -X GET "$GRID_BASE_URL/internal-accounts?customerId=$CUSTOMER_ID" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" | jq -r '.data[0].id')
curl -s -X POST "$GRID_BASE_URL/sandbox/internal-accounts/$ACCOUNT/fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"amount": "10000", "currency": "USD"}'

# 3. Add UPI beneficiary
echo "Adding beneficiary..."
EXT_ACCOUNT=$(curl -s -X POST "$GRID_BASE_URL/customers/$CUSTOMER_ID/external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "UPI",
    "upiId": "test@upi",
    "currency": "INR",
    "accountOwnerName": "Test Recipient"
  }' | jq -r '.id')

# 4. Create quote
echo "Creating quote..."
QUOTE=$(curl -s -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d "{
    "source": {"sourceType": "ACCOUNT", "accountId": "$ACCOUNT"},
    "destination": {"destinationType": "ACCOUNT", "accountId": "$EXT_ACCOUNT"},
    "amount": "100.00",
    "amountCurrency": "USD"
  }")
QUOTE_ID=$(echo $QUOTE | jq -r '.id')
echo "Quote: $QUOTE_ID"

# 5. Execute quote
echo "Executing quote..."
TX=$(curl -s -X POST "$GRID_BASE_URL/quotes/$QUOTE_ID/execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET")
echo "Transaction: $(echo $TX | jq -r '.id')"
echo "Status: $(echo $TX | jq -r '.status')"

echo "Test complete!"
```

---

## Step 9: Production Deployment

### Checklist

- [ ] All sandbox tests passing
- [ ] KYC flow tested end-to-end
- [ ] Webhook handler verified
- [ ] Error handling implemented
- [ ] Idempotency keys used
- [ ] Logging configured
- [ ] Monitoring set up
- [ ] Rate limiting understood
- [ ] Support contacts established
- [ ] Compliance review complete

### Go-Live Steps

1. Request production credentials from Lightspark
2. Configure production webhooks
3. Set up production monitoring
4. Start with limited users (beta)
5. Monitor closely
6. Gradually scale up

---

## Real-World Example: Complete US to India Transfer

### Scenario

**Sender:** Raj Patel in San Francisco, CA
- Wants to send $500 to his family in Mumbai, India
- Has completed KYC
- Has $5,000 in his USD account

**Recipient:** Family member in Mumbai
- UPI ID: `family@okicici`
- Will receive Rs. 41,500

### Complete API Sequence

```bash
# 1. Get customer's accounts
curl -X GET "$GRID_BASE_URL/internal-accounts?customerId=Customer:raj-patel-id" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
# -> InternalAccount:usd-raj (balance: $5000)

# 2. Get recipient's external account
curl -X GET "$GRID_BASE_URL/customers/Customer:raj-patel-id/external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
# -> ExternalAccount:upi-family

# 3. Create quote
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {"sourceType": "ACCOUNT", "accountId": "InternalAccount:usd-raj"},
    "destination": {"destinationType": "ACCOUNT", "accountId": "ExternalAccount:upi-family"},
    "amount": "500.00",
    "amountCurrency": "USD"
  }'
# -> Quote:quote-001 (rate: 83.00, fee: $7.50, total: $507.50)

# 4. Execute
curl -X POST "$GRID_BASE_URL/quotes/Quote:quote-001/execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Idempotency-Key: raj-20250115-001"
# -> Transaction:tx-001 (status: PENDING)

# 5. Webhooks received:
#    - transaction.status_change: PENDING -> IN_FLIGHT (instant)
#    - transaction.status_change: IN_FLIGHT -> COMPLETED (~2 minutes)

# 6. Family receives Rs. 41,500 via UPI
```

### Timeline

| Time | Event |
|------|-------|
| T+0s | Quote created, rate locked |
| T+5s | Customer confirms, quote executed |
| T+10s | Transaction PENDING -> IN_FLIGHT |
| T+30s | USD converted to BTC via Lightning |
| T+60s | BTC converted to INR |
| T+90s | INR sent via UPI |
| T+120s | Transaction COMPLETED |
| T+125s | Customer receives notification |

---

## SumSub Integration Deep Dive

### SumSub Configuration

Grid uses SumSub for KYC/KYB. You don't need a separate SumSub account - Grid handles the integration.

### KYC Flow Steps

1. **Customer Data Collection**
   - Collect full name, DOB, address, nationality
   - Collect government ID number (SSN, passport, etc.)

2. **Generate KYC Link**
   ```bash
   POST /customers/{id}/kyc-link
   ```

3. **Redirect to SumSub**
   - Customer completes on SumSub's hosted page
   - Documents uploaded directly to SumSub
   - Liveness check performed

4. **SumSub Reviews**
   - Automated checks (OCR, face matching, sanctions)
   - Manual review if needed
   - Decision: APPROVED, REJECTED, or MORE_INFO

5. **Grid Receives Result**
   - SumSub notifies Grid
   - Grid updates customer status
   - Grid sends webhook to your platform

6. **Your Platform Acts**
   - Enable/disable features based on KYC status
   - Notify customer

### SumSub SDK Integration (Optional)

If you want to embed KYC in your app instead of redirecting:

```javascript
// 1. Generate KYC link to get token
const { token } = await grid.customers.createKycLink(customerId);

// 2. Embed SumSub Web SDK
import snsWebSdk from '@sumsub/websdk';

const sdk = snsWebSdk.init(token, () => refreshToken())
  .withConf({
    lang: 'en',
    theme: 'light',
  })
  .on('idCheck.onFinished', (payload) => {
    console.log('KYC completed');
  })
  .build();

sdk.launch('#kyc-container');
```

### Required Documents by Country

| Country | ID | Proof of Address |
|---------|----|------------------|
| US | Passport, Driver's License, State ID | Utility bill, Bank statement |
| India | Passport, Aadhaar, PAN | Utility bill, Rental agreement |
| UK | Passport, Driver's License, BRP | Utility bill, Council tax bill |
| EU | Passport, National ID | Utility bill, Bank statement |

### Handling KYC Errors

```json
{
  "verificationStatus": "RESOLVE_ERRORS",
  "errors": [
    {
      "type": "MISSING_FIELD",
      "field": "address.line1",
      "reason": "Address line 1 is required"
    },
    {
      "type": "MISSING_PROOF_OF_ADDRESS_DOCUMENT",
      "reason": "Proof of address document is required"
    }
  ]
}
```

Show these errors to the customer and allow them to re-submit.

---

## API Quick Reference

### Authentication
```bash
curl -u "CLIENT_ID:CLIENT_SECRET" https://api.lightspark.com/grid/2025-10-13/...
```

### Key Endpoints

| Operation | Endpoint | Method |
|-----------|----------|--------|
| Create Customer | `/customers` | POST |
| Get Customer | `/customers/{id}` | GET |
| Generate KYC Link | `/customers/{id}/kyc-link` | POST |
| List Internal Accounts | `/internal-accounts` | GET |
| Add External Account | `/customers/{id}/external-accounts` | POST |
| Create Quote | `/quotes` | POST |
| Execute Quote | `/quotes/{id}/execute` | POST |
| List Transactions | `/transactions` | GET |
| Get Exchange Rates | `/exchange-rates` | GET |
| List Banks | `/discoveries` | GET |

### Sandbox Magic Values

| Context | Value | Result |
|---------|-------|--------|
| KYC lastName suffix | `001` | PENDING |
| KYC lastName suffix | `002` | REJECTED |
| KYC lastName suffix | anything else | APPROVED |
| KYB registrationNumber suffix | `001` | PENDING |
| KYB registrationNumber suffix | `002` | REJECTED |
| Email OTP | `000000` | Accepted |
| Wallet Signature | any base64 | Accepted |

---

## Troubleshooting Guide

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 401 Unauthorized | Wrong credentials | Check CLIENT_ID and CLIENT_SECRET |
| 404 Not Found | Wrong resource ID | Verify ID format (Customer:...) |
| Quote expired | Waited too long | Create new quote |
| KYC pending | Using magic suffix 001 | Use different suffix or wait |
| Insufficient funds | Account balance low | Fund account first |
| Bank rejected | Invalid account details | Verify account number |
| Webhook not received | URL not accessible | Check HTTPS, firewall |
| Rate limited | Too many requests | Implement backoff |

### Getting Help

- **Documentation:** https://docs.lightspark.com
- **GitHub Issues:** https://github.com/lightsparkdev/grid-api/issues
- **Contact:** https://www.lightspark.com/contact
- **Dashboard:** https://app.lightspark.com

### Debug Tips

1. **Use sandbox** - Test everything in sandbox first
2. **Check logs** - Log all API requests and responses
3. **Verify webhooks** - Use ngrok for local webhook testing
4. **Monitor dashboard** - Check Grid dashboard for status
5. **Test magic values** - Use magic values to simulate scenarios

---

*This guide covers the complete Lightspark Grid integration. For detailed API documentation, refer to the individual files in this documentation folder.*
