# SumSub Integration Guide

Complete guide to SumSub identity verification integration through Grid.

## What is SumSub?

SumSub is a leading identity verification platform that Grid uses for KYC (Know Your Customer) and KYB (Know Your Business) verification. SumSub provides:

- Document verification (passports, driver's licenses, national IDs)
- Liveness detection and biometric matching
- Address verification
- AML screening (sanctions, PEP, adverse media)
- Business verification
- Ongoing monitoring

## How Grid Uses SumSub

Grid has integrated SumSub as the default KYC/KYB provider. When you use Grid's hosted KYC flow:

1. You call Grid's API to generate a KYC link
2. Grid creates a SumSub applicant profile
3. Customer completes verification on SumSub's hosted page
4. SumSub notifies Grid of the result
5. Grid updates the customer's verification status
6. Grid sends you a webhook notification

## Integration Options

### Option 1: Grid Hosted Flow (Recommended)

The simplest integration. You redirect customers to SumSub's hosted page.

**Pros:**
- Minimal implementation
- Grid handles all SumSub configuration
- Automatic status updates
- Webhook notifications

**Cons:**
- Less control over UI
- Customer leaves your domain

**Implementation:**
```bash
# 1. Generate KYC link
POST /customers/{id}/kyc-link

# 2. Redirect customer to the returned URL
# 3. Handle webhook when complete
```

### Option 2: Embedded Flow (Advanced)

Embed SumSub's verification widget directly in your application.

**Pros:**
- Customer stays in your app
- Full control over styling
- Better user experience

**Cons:**
- More complex implementation
- Need SumSub SDK
- Handle token refresh

**Implementation:**
```javascript
// 1. Generate KYC link to get token
const { token } = await grid.customers.createKycLink(customerId);

// 2. Embed SumSub SDK
import snsWebSdk from '@sumsub/websdk';

const sdk = snsWebSdk.init(token, refreshToken)
  .withConf({ lang: 'en' })
  .build();

sdk.launch('#kyc-container');
```

### Option 3: Bring Your Own KYC

Use your existing KYC provider instead of SumSub.

**Requirements:**
- Contact Lightspark to discuss integration
- Provide webhook endpoint for status updates
- Meet compliance requirements

## SumSub SDK Installation

### Web SDK

```bash
npm install @sumsub/websdk
```

```javascript
import snsWebSdk from '@sumsub/websdk';

function launchKyc(accessToken, containerId) {
  const sdk = snsWebSdk.init(accessToken, () => getNewToken())
    .withConf({
      lang: 'en',
      theme: 'light',
      email: 'customer@example.com',
      phone: '+1234567890',
    })
    .withOptions({ addViewportTag: false, adaptIframeHeight: true })
    .build();

  sdk.launch(`#${containerId}`);

  // Event handlers
  sdk.on('idCheck.onApplicantLoaded', (payload) => {
    console.log('Applicant loaded:', payload);
  });

  sdk.on('idCheck.onStepCompleted', (payload) => {
    console.log('Step completed:', payload);
  });

  sdk.on('idCheck.onFinished', (payload) => {
    console.log('KYC finished:', payload);
    // Check verification status via API
  });

  sdk.on('idCheck.onError', (payload) => {
    console.error('KYC error:', payload);
  });

  return sdk;
}
```

### Mobile SDKs

SumSub provides SDKs for iOS and Android:

**iOS (Swift):**
```swift
import IdensicMobileSDK

let config = SNSMobileSDKConfiguration(
    accessToken: token,
    environment: .production
)

let sdk = SNSMobileSDK(configuration: config)
sdk.tokenExpirationHandler { callback in
    // Refresh token
    getNewToken { newToken in
        callback(newToken)
    }
}

sdk.present()
```

**Android (Kotlin):**
```kotlin
val sdk = SNSMobileSDK.Builder(this, token)
    .withEnvironment(Environment.PRODUCTION)
    .withCallbacks(object : SNSEventListener {
        override fun onEvent(event: SNSEvent, sdk: SNSMobileSDK) {
            when (event) {
                is SNSEvent.SDKLaunched -> { }
                is SNSEvent.SDKClosed -> { }
                is SNSEvent.StepCompleted -> { }
                is SNSEvent.SDKFinished -> { }
            }
        }
    })
    .build()

sdk.launch()
```

## SumSub Verification Flow

### Step 1: Agreement
Customer accepts terms and conditions.

### Step 2: Document Upload
Customer uploads government-issued ID:
- Passport
- Driver's license
- National ID card

Requirements:
- All corners visible
- Not expired
- Clear, not blurry
- No glare

### Step 3: Liveness Check
Customer takes a selfie or completes a liveness detection:
- Face clearly visible
- Good lighting
- No glasses (recommended)
- Follow on-screen instructions

### Step 4: Address Verification (if required)
Upload proof of address:
- Utility bill (electricity, water, gas)
- Bank statement
- Lease agreement
- Official government correspondence

Must be:
- Dated within last 3 months
- Show customer's name and address
- Match the address on file

### Step 5: Review
SumSub reviews submitted documents:
- Automated checks: seconds to minutes
- Manual review: hours to 1-2 business days
- Complex cases: up to 5 business days

### Step 6: Result
Customer receives approval or rejection.

## Verification Levels

SumSub supports different verification levels:

| Level | Checks | Typical Use Case |
|-------|--------|-----------------|
| Basic | Document + selfie | Low-risk transactions |
| Standard | Document + liveness + address | Standard accounts |
| Enhanced | All above + AML screening | High-value accounts |
| Corporate | Business docs + beneficial owners | Business accounts |

Grid uses Standard level by default. Contact Lightspark for custom levels.

## Supported Documents by Country

### Tier 1 (Full Support)
- US, UK, EU countries, Canada, Australia, Japan, Singapore
- All major ID types supported
- Address verification supported

### Tier 2 (Good Support)
- Most of Asia, Latin America, Middle East
- Passports and national IDs
- Some local ID types

### Tier 3 (Basic Support)
- Remaining countries
- Passports primarily
- May require manual review

## AML Screening

SumSub performs automatic AML screening:

### Sanctions Screening
- UN, OFAC, EU, HMT sanctions lists
- Updated daily
- Real-time screening

### PEP Screening
- Politically Exposed Persons
- Family members and close associates
- Ongoing monitoring

### Adverse Media
- Negative news screening
- Risk-based approach
- Configurable sensitivity

## Ongoing Monitoring

After initial verification, SumSub provides:

### Re-verification Triggers
- Document expiration
- Change in customer information
- Suspicious activity
- Regulatory requirements

### Continuous Screening
- Daily sanctions list updates
- Ongoing PEP monitoring
- Adverse media alerts

## Customization Options

### Branding
Customize the look and feel:
- Logo
- Primary color
- Welcome message
- Instructions

Configure via Grid dashboard or SumSub admin panel.

### Localization
Supported languages:
- English (default)
- Spanish
- French
- German
- Portuguese
- And 30+ more

Set via `lang` parameter in SDK configuration.

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Document rejected | Blurry/cropped | Retake with better lighting |
| Liveness failed | Poor lighting | Use natural light, no backlight |
| Name mismatch | Typo in application | Update customer record |
| Pending too long | Manual review | Wait 1-2 business days |
| Token expired | Timeout | Generate new KYC link |

### Error Codes

| Code | Meaning | Action |
|------|---------|--------|
| `INVALID_TOKEN` | KYC link expired | Generate new link |
| `APPLICANT_NOT_FOUND` | SumSub profile missing | Contact support |
| `VERIFICATION_FAILED` | Technical error | Retry or contact support |

## Compliance Notes

### Data Handling
- SumSub stores documents securely
- GDPR compliant
- Data retention per regulatory requirements
- Customer can request data deletion

### Regulatory Compliance
- AML directives (EU, US, etc.)
- CDD/EDD requirements
- Record keeping requirements
- Audit trail maintained

## Support

For SumSub-specific issues:
- Grid Support: https://www.lightspark.com/contact
- SumSub Docs: https://docs.sumsub.com/
- SumSub Support: support@sumsub.com
