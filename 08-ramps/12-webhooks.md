# Ramps Webhooks

Receive real-time notifications for ramp conversions, account updates, and transaction status.

Same webhook events as [Payouts webhooks](../07-payouts-and-b2b/14-webhooks.md).

Key events for Ramps:
- `transaction.status_change` - Conversion completed/failed
- `incoming_payment.received` - Crypto/fiat deposit received
- `internal_account.status_change` - Balance updates
