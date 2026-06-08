# Verification Status Change Webhook

Webhook that is called when a customer's KYC/KYB verification status changes.

## Event Type

```
verification.status_change
```

## Payload

```json
{
  "eventType": "verification.status_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "verificationId": "Verification:019542f5-b3e7-1d02-0000-000000000001",
    "customerId": "Customer:019542f5-b3e7-1d02-0000-000000000001",
    "status": "APPROVED",
    "previousStatus": "PENDING",
    "kycStatus": "APPROVED",
    "kybStatus": null,
    "errors": [],
    "completedAt": "2025-01-15T10:30:00Z"
  }
}
```

## Payload Fields

| Field | Type | Description |
|-------|------|-------------|
| `verificationId` | string | The verification record ID |
| `customerId` | string | The customer being verified |
| `status` | string | New verification status |
| `previousStatus` | string | Previous status |
| `kycStatus` | string | Individual KYC status (if applicable) |
| `kybStatus` | string | Business KYB status (if applicable) |
| `errors` | array | Any errors requiring resolution |
| `completedAt` | string | Completion timestamp |

## Status Values

| Status | Meaning |
|--------|---------|
| `RESOLVE_ERRORS` | Missing information, needs correction |
| `PENDING` | Submitted, awaiting review |
| `IN_REVIEW` | Under manual review |
| `APPROVED` | Verification successful |
| `REJECTED` | Verification rejected |

## Handling the Webhook

### Example Handler (Node.js)

```javascript
app.post('/webhooks/grid', async (req, res) => {
  const { eventType, data } = req.body;

  if (eventType === 'verification.status_change') {
    const { customerId, status, kycStatus } = data;

    // Update customer in your database
    await db.customers.update(customerId, {
      kycStatus: kycStatus,
      verificationStatus: status,
      verifiedAt: status === 'APPROVED' ? new Date() : null
    });

    // Notify customer
    if (status === 'APPROVED') {
      await notifyCustomer(customerId, 'Your verification is complete!');
    } else if (status === 'REJECTED') {
      await notifyCustomer(customerId, 'Verification failed. Please contact support.');
    } else if (status === 'RESOLVE_ERRORS') {
      await notifyCustomer(customerId, 'Additional information needed.');
    }
  }

  res.status(200).send('OK');
});
```

## Common Status Transitions

### Individual KYC

```
NOT_STARTED -> PENDING -> APPROVED
                        -> REJECTED
                        -> RESOLVE_ERRORS -> PENDING -> APPROVED
```

### Business KYB

```
NOT_STARTED -> PENDING -> IN_REVIEW -> APPROVED
                                    -> REJECTED
                                    -> RESOLVE_ERRORS -> IN_REVIEW -> APPROVED
```

## Actions by Status

| Status | Action to Take |
|--------|---------------|
| `APPROVED` | Enable customer's payment features |
| `REJECTED` | Restrict account, contact customer |
| `RESOLVE_ERRORS` | Show customer what's missing |
| `PENDING` | Wait for review completion |
| `IN_REVIEW` | No action needed |

## Best Practices

1. **Process asynchronously** - Return 200 quickly, process in background
2. **Update customer record** - Store latest status in your database
3. **Notify customer** - Send email/push notification on status change
4. **Handle RESOLVE_ERRORS** - Show specific errors to customer
5. **Log transitions** - Keep audit log of all status changes
6. **Enable features conditionally** - Only enable payments after APPROVED
