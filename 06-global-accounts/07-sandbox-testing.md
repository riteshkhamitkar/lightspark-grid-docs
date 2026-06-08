# Global Account Sandbox Testing

Exercise the Global Account auth, signing, and funding flows without standing up real OTP delivery, WebAuthn, or OIDC providers.

## Sandbox Features

### Automatic KYC
Customers are automatically KYC-approved on creation in sandbox.

### Auto-Provisioned Accounts
Global Accounts (USDB) are created automatically when you create a customer.

### Simulated Authentication

| Credential Type | Sandbox Behavior |
|----------------|------------------|
| PASSKEY | WebAuthn ceremonies auto-approved |
| EMAIL_OTP | OTP code is always `000000` |
| OAUTH | Any valid JWT format accepted |

### Wallet Signing

Any valid base64 string is accepted as a `Grid-Wallet-Signature` header in sandbox.

## Quick Test Flow

### 1. Create Customer

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "test-001",
    "region": "US",
    "email": "test@example.com",
    "fullName": "Test User"
  }'
```

Customer created with KYC APPROVED.

### 2. Find Global Account

```bash
curl -X GET "$GRID_BASE_URL/internal-accounts?customerId=Customer:..." \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

Account with `type: "EMBEDDED_WALLET"` and `currency: "USDB"`.

### 3. Fund Account

```bash
curl -X POST "$GRID_BASE_URL/sandbox/internal-accounts/InternalAccount:.../fund" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"amount": "1000.00", "currency": "USDB"}'
```

### 4. Create Credential (Email OTP)

```bash
curl -X POST "$GRID_BASE_URL/auth/credentials" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "internalAccountId": "InternalAccount:...",
    "credentialType": "EMAIL_OTP",
    "name": "test@example.com"
  }'
```

### 5. Verify with OTP

```bash
curl -X POST "$GRID_BASE_URL/auth/credentials/{credentialId}/verify" \
  -H "Content-Type: application/json" \
  -d '{"otpCode": "000000"}'
```

### 6. Add External Account

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "ACH",
    "accountNumber": "123456789",
    "routingNumber": "021000021",
    "currency": "USD",
    "accountOwnerName": "Test User"
  }'
```

### 7. Create Withdrawal Quote

```bash
curl -X POST "$GRID_BASE_URL/quotes" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {"sourceType": "ACCOUNT", "accountId": "InternalAccount:ga-..."},
    "destination": {"destinationType": "ACCOUNT", "accountId": "ExternalAccount:ach-..."},
    "amount": "100.00",
    "amountCurrency": "USDB"
  }'
```

### 8. Execute with Dummy Signature

```bash
curl -X POST "$GRID_BASE_URL/quotes/Quote:.../execute" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -H "Grid-Wallet-Signature: dGVzdF9zaWduYXR1cmU="  # any base64 string
```

## Testing Checklist

- [ ] Customer creation with auto-provisioned Global Account
- [ ] KYC auto-approved in sandbox
- [ ] Passkey registration (auto-completed)
- [ ] Email OTP verification (code: 000000)
- [ ] OAuth flow (any JWT)
- [ ] Account funding via sandbox endpoint
- [ ] Withdrawal quote creation
- [ ] Quote execution with signature
- [ ] Transaction status webhooks
- [ ] Session management (list/revoke)
- [ ] Credential management (list/revoke)

## Magic Values Reference

| Context | Value | Result |
|---------|-------|--------|
| Email OTP | `000000` | Always accepted |
| WebAuthn | Any response | Auto-approved |
| Wallet Signature | Any base64 | Accepted |
| OAuth Token | Any JWT format | Accepted |

## Limitations

- No real money movement
- No real bank settlements
- WebAuthn is simulated (no actual biometric)
- OTP is hardcoded
- Network delays are simulated, not real
