# Error Handling

Handle payment failures, API errors, and transaction issues in Global P2P.

Same as [Payouts error handling](../07-payouts-and-b2b/12-error-handling.md).

Additional P2P errors:

| Error | Cause | Solution |
|-------|-------|----------|
| `UMA_NOT_FOUND` | UMA address doesn't exist | Verify address |
| `UMA_UNAVAILABLE` | UMA service down | Retry later |
| `INVALID_UMA` | Malformed UMA address | Check format |
| `RECEIVER_REJECTED` | Receiver declined | Contact receiver |
