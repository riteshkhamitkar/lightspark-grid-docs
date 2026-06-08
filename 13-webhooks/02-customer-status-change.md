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

## Actions

| New Status | Action |
|-----------|--------|
| `APPROVED` | Enable payment features |
| `REJECTED` | Restrict account |
| `RESOLVE_ERRORS` | Show missing info |
| `PENDING` | Wait for review |
