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
    "verificationId": "Verification:...",
    "customerId": "Customer:...",
    "status": "APPROVED",
    "previousStatus": "PENDING",
    "kycStatus": "APPROVED",
    "errors": [],
    "completedAt": "2025-01-15T10:30:00Z"
  }
}
```

## Handling

1. Update customer verification status in your database
2. Enable features if APPROVED
3. Notify customer of status change
4. Handle RESOLVE_ERRORS by showing missing info
