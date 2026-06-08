# UMA Invitations

Invite users to join your platform via UMA.

## Create Invitation

```bash
POST /uma/invitations
```

```bash
curl -X POST "$GRID_BASE_URL/uma/invitations" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "inviterCustomerId": "Customer:sender-001",
    "inviteeEmail": "friend@example.com",
    "message": "Join me on YourApp for instant money transfers!"
  }'
```

## Claim Invitation

```bash
POST /uma/invitations/{code}/claim
```

```bash
curl -X POST "$GRID_BASE_URL/uma/invitations/INVITE123/claim" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "inviteeCustomerId": "Customer:new-user-002"
  }'
```

## Get Invitation

```bash
GET /uma/invitations/{code}
```

## Cancel Invitation

```bash
DELETE /uma/invitations/{code}
```

## Webhook Event

```json
{
  "eventType": "invitation.claimed",
  "data": {
    "invitationCode": "INVITE123",
    "inviterId": "Customer:sender-001",
    "inviteeId": "Customer:new-user-002"
  }
}
```

## Best Practices

1. **Track invitations** - Monitor claim rates
2. **Reward referrals** - Incentivize invitations
3. **Expiration** - Set expiration on invitations
4. **Personalize message** - Custom invitation text
