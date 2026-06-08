# Cards Sandbox Testing

Drive deterministic card outcomes with the simulate endpoints.

## Simulate Card Authorization

```bash
POST /sandbox/cards/{card_id}/authorize
```

```bash
curl -X POST "$GRID_BASE_URL/sandbox/cards/Card:.../authorize" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "50.00",
    "currency": "USD",
    "merchantName": "Test Store",
    "mcc": "5411"
  }'
```

## Simulate Card Clearing

```bash
POST /sandbox/cards/{card_id}/clear
```

```bash
curl -X POST "$GRID_BASE_URL/sandbox/cards/Card:.../clear" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "authorizationId": "Auth:...",
    "amount": "50.00"
  }'
```

## Simulate Card Return (Refund)

```bash
POST /sandbox/cards/{card_id}/return
```

```bash
curl -X POST "$GRID_BASE_URL/sandbox/cards/Card:.../return" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "clearingId": "Clearing:...",
    "amount": "50.00"
  }'
```

## Magic Values

| Amount Suffix | Result |
|--------------|--------|
| `.00` | Approval |
| `.01` | Decline - insufficient funds |
| `.02` | Decline - card frozen |
| `.03` | Decline - invalid merchant |

## Testing Checklist

- [ ] Issue card
- [ ] Card lifecycle (PENDING_ISSUE -> ACTIVE)
- [ ] Authorization simulation
- [ ] Clearing simulation
- [ ] Refund simulation
- [ ] Freeze/unfreeze
- [ ] Close card
- [ ] Webhook events
- [ ] Funding source management
