# Customer Status Change Webhook

Webhook that is called when the status of a customer is updated, including KYC and KYB status changes.

## Event Type

```
customer.status_change
```

## Payload

```json
{
  "eventType": "customer.status_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "customerId": "Customer:019542f5-b3e7-1d02-0000-000000000001",
    "previousStatus": {
      "kycStatus": "PENDING"
    },
    "newStatus": {
      "kycStatus": "APPROVED"
    },
    "changedFields": ["kycStatus"],
    "updatedAt": "2025-01-15T10:30:00Z"
  }
}
```

## Payload Fields

| Field | Type | Description |
|-------|------|-------------|
| `customerId` | string | The customer whose status changed |
| `previousStatus` | object | Previous status values |
| `newStatus` | object | New status values |
| `changedFields` | string[] | Which fields changed |
| `updatedAt` | string | When the change occurred |

## Status Fields That Trigger This Webhook

- `kycStatus` - KYC verification status
- `kybStatus` - KYB verification status
- `accountStatus` - Overall account status
- `complianceStatus` - Compliance hold status

## Handling the Webhook

### Example Handler (Node.js)

```javascript
app.post('/webhooks/grid', async (req, res) => {
  const { eventType, data } = req.body;

  if (eventType === 'customer.status_change') {
    const { customerId, newStatus, changedFields } = data;

    // Update customer in database
    await db.customers.update(customerId, {
      kycStatus: newStatus.kycStatus,
      updatedAt: new Date()
    });

    // Handle KYC approval
    if (changedFields.includes('kycStatus') && newStatus.kycStatus === 'APPROVED') {
      // Enable payment features
      await enablePayments(customerId);

      // Notify customer
      await sendEmail(customerId, 'Your account is verified!');
    }

    // Handle KYC rejection
    if (changedFields.includes('kycStatus') && newStatus.kycStatus === 'REJECTED') {
      // Restrict account
      await restrictAccount(customerId);

      // Notify customer
      await sendEmail(customerId, 'Verification failed. Contact support.');
    }
  }

  res.status(200).send('OK');
});
```

## Common Status Changes

### Individual Customer

| From | To | Meaning |
|------|-----|---------|
| `NOT_STARTED` | `PENDING` | KYC initiated |
| `PENDING` | `IN_REVIEW` | Documents under review |
| `IN_REVIEW` | `APPROVED` | Verification successful |
| `IN_REVIEW` | `REJECTED` | Verification failed |
| `APPROVED` | `RESTRICTED` | Compliance issue detected |

### Business Customer

| From | To | Meaning |
|------|-----|---------|
| `NOT_STARTED` | `PENDING` | KYB initiated |
| `PENDING` | `IN_REVIEW` | Under review |
| `IN_REVIEW` | `APPROVED` | All checks passed |
| `IN_REVIEW` | `RESOLVE_ERRORS` | Missing information |
| `RESOLVE_ERRORS` | `IN_REVIEW` | Resubmitted |
| `IN_REVIEW` | `REJECTED` | Verification failed |

## Best Practices

1. **Enable features conditionally** - Only enable payments after APPROVED
2. **Notify customers** - Send email/push on status changes
3. **Log changes** - Keep audit trail
4. **Handle edge cases** - RESTRICTED, SUSPENDED states
5. **Process async** - Return 200 quickly, process in background
