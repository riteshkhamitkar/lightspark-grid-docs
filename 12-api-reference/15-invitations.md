# Invitations API (UMA)

## Create a UMA Invitation

```bash
POST /uma/invitations
```

```bash
curl -X POST "$GRID_BASE_URL/uma/invitations" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "inviterCustomerId": "Customer:...",
    "inviteeEmail": "friend@example.com"
  }'
```

## Get a UMA Invitation by Code

```bash
GET /uma/invitations/{code}
```

## Claim a UMA Invitation

```bash
POST /uma/invitations/{code}/claim
```

```bash
curl -X POST "$GRID_BASE_URL/uma/invitations/{code}/claim" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "inviteeCustomerId": "Customer:..."
  }'
```

## Cancel a UMA Invitation

```bash
DELETE /uma/invitations/{code}
```
