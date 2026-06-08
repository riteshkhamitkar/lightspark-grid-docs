# Invitation Claimed Webhook

Webhook that is called when an invitation is claimed by a customer.

## Event Type

```
invitation.claimed
```

## Payload

```json
{
  "eventType": "invitation.claimed",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "invitationCode": "INVITE123",
    "inviterId": "Customer:...",
    "inviteeId": "Customer:..."
  }
}
```

## Handling

1. Credit referral bonus (if applicable)
2. Notify inviter
3. Update referral tracking
4. Welcome new user
