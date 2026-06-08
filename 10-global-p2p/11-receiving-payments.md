# Receiving Payments

Receiving payments from UMA addresses.

## How It Works

1. Customer has UMA address: `$user@your-domain.com`
2. Sender creates quote to this UMA address
3. Grid routes payment to customer's internal account
4. Customer receives funds (converted to their currency if needed)

## Webhook Notification

```json
{
  "eventType": "incoming_payment.received",
  "data": {
    "customerId": "Customer:...",
    "accountId": "InternalAccount:...",
    "amount": "100.00",
    "currency": "USD",
    "sourceUmaAddress": "$sender@other-domain.com"
  }
}
```

## Displaying Received Payments

```bash
curl -X GET "$GRID_BASE_URL/transactions?customerId=Customer:...&type=TRANSFER_IN" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET"
```

## Sharing UMA Address

Provide customers with their UMA address:
- Display in app
- Share via QR code
- Copy to clipboard
- Share via messaging

## Auto-Conversion

If payment currency differs from account currency, Grid can auto-convert:
- USD payment to EUR account
- BTC payment to USD account
- Configure in platform settings
