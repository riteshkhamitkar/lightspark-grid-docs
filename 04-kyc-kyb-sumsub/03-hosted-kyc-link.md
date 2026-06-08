# Generate a Hosted KYC Link

Generate a single-use hosted URL the customer can complete to verify their identity.

## API Endpoint

```bash
POST /customers/{customerId}/kyc-link
```

## Request

```bash
curl --request POST \
  --url https://api.lightspark.com/grid/2025-10-13/customers/Customer:.../kyc-link \
  --header 'Authorization: Basic <encoded-value>' \
  --header 'Content-Type: application/json'
```

## Response

```json
{
  "url": "https://verify.sumsub.com/idensic/liveness/#/uni?_si=...",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "expiresAt": "2025-01-15T11:00:00Z"
}
```

## Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `url` | string | The hosted KYC URL to redirect the customer to |
| `token` | string | Provider-specific token for embedding via SDK |
| `expiresAt` | string | URL expiration timestamp (typically 24 hours) |

## Using the KYC Link

### Redirect Method (Recommended)

1. Generate the KYC link via API
2. Redirect customer browser to the `url`
3. Customer completes verification on SumSub's page
4. Customer is redirected back to your app
5. You receive webhook notification when complete

```javascript
// Backend
const kycLink = await grid.customers.createKycLink(customerId);

// Frontend
window.location.href = kycLink.url;
```

### Embedded Method (Advanced)

Use the `token` with SumSub's SDK to embed the verification flow directly in your app:

```javascript
import snsWebSdk from '@sumsub/websdk';

const launchWebSdk = (token) => {
  const snsWebSdkInstance = snsWebSdk
    .init(token, () => getNewToken())
    .withConf({
      lang: 'en',
      email: 'customer@example.com',
      phone: '+1234567890',
    })
    .build();

  snsWebSdkInstance.launch('#sumsub-websdk-container');
};
```

## KYC Flow Steps

The hosted KYC flow typically includes:

1. **Agreement** - Customer agrees to terms
2. **Document Upload** - Upload government-issued ID
  - Passport
  - Driver's license
  - National ID card
3. **Selfie/Liveness** - Take a selfie or complete liveness check
4. **Address Verification** - Upload proof of address (if required)
5. **Review** - System reviews submitted documents
6. **Result** - Approval or rejection

## Redirect URL Configuration

Configure redirect URLs in your platform settings or pass them when generating the link:

```bash
curl --request POST \
  --url https://api.lightspark.com/grid/2025-10-13/customers/Customer:.../kyc-link \
  --header 'Authorization: Basic <encoded-value>' \
  --header 'Content-Type: application/json' \
  --data '{
    "redirectUrl": "https://your-app.com/kyc-complete?customerId=..."
  }'
```

## Token Expiration

- KYC links expire after 24 hours
- Generate a new link if the old one expires
- Each link is single-use

## Handling Completion

After customer completes KYC:

1. **Webhook notification** (recommended):
   - Listen for `verification.status_change` webhook
   - Update customer status in your system

2. **Polling** (alternative):
   - Poll `GET /verifications/{id}` periodically
   - Check `verificationStatus` field

3. **Redirect**:
   - Customer redirected to your `redirectUrl`
   - Check status via API when page loads

## Best Practices

1. **Generate links on-demand** - Don't pre-generate; create when customer is ready
2. **Handle expiration** - Generate new link if expired
3. **Clear instructions** - Tell customer what documents to prepare
4. **Mobile-friendly** - Most customers complete on mobile
5. **Support multiple languages** - SumSub supports many languages
6. **Test the flow** - Complete the KYC flow yourself in sandbox

## SumSub SDK Integration

For embedded integration, you'll need:

```bash
npm install @sumsub/websdk
```

```javascript
import snsWebSdk from '@sumsub/websdk';

// Initialize with token from Grid API
const sdk = snsWebSdk.init(token, refreshTokenCallback)
  .withConf({
    lang: 'en',
    theme: 'light',
  })
  .build();

sdk.launch('#kyc-container');

// Handle events
sdk.on('idCheck.onApplicantLoaded', (payload) => {
  console.log('Applicant loaded', payload);
});

sdk.on('idCheck.onFinished', (payload) => {
  console.log('KYC finished', payload);
  // Check verification status via API
});
```

## Supported Documents by Country

| Country | Supported IDs | Notes |
|---------|--------------|-------|
| US | Passport, Driver's License, State ID | SSN required |
| UK | Passport, Driver's License, National ID | Proof of address required |
| EU | Passport, National ID, Driver's License | Varies by country |
| Canada | Passport, Driver's License, PR Card | |
| Australia | Passport, Driver's License | |
| Other | Passport | May require additional docs |

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Link expired | Generate new link |
| Upload failed | Check file format (JPG, PNG, PDF) and size (< 10MB) |
| Liveness failed | Ensure good lighting, clear camera, no glasses |
| Document rejected | Ensure clear photo, all corners visible, not expired |
