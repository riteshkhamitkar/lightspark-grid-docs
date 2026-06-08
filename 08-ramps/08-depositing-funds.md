# Depositing Funds for Ramps

Depositing funds into internal accounts for ramp operations.

Same as [Payouts depositing funds](../07-payouts-and-b2b/08-depositing-funds.md).

For crypto deposits:
- Send crypto to the internal account's deposit address
- Monitor via `incoming_payment.received` webhook
- Confirm on-chain before crediting
