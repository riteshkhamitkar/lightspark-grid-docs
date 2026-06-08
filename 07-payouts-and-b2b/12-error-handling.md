# Error Handling

Handle payment failures, API errors, and transaction issues gracefully.

## HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success | Process response |
| 201 | Created | Resource created successfully |
| 400 | Bad Request | Check request parameters |
| 401 | Unauthorized | Check API credentials |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource already exists |
| 422 | Unprocessable | Business logic error |
| 429 | Rate Limited | Slow down requests |
| 500 | Server Error | Retry with backoff |
| 502 | Bad Gateway | Retry with backoff |
| 503 | Service Unavailable | Retry with backoff |

## Common Error Codes

### Quote Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `QUOTE_EXPIRED` | Quote timeout | Create new quote |
| `INSUFFICIENT_FUNDS` | Low balance | Fund account |
| `INVALID_ACCOUNT` | Account closed | Check account status |
| `UNSUPPORTED_CORRIDOR` | Currency pair not supported | Check supported currencies |
| `AMOUNT_TOO_SMALL` | Below minimum | Increase amount |
| `AMOUNT_TOO_LARGE` | Above maximum | Decrease amount or request limit increase |
| `RATE_UNAVAILABLE` | FX rate not available | Retry later |

### Transfer Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `BANK_REJECTED` | Bank returned payment | Verify account details |
| `BENEFICIARY_MISMATCH` | Name doesn't match account | Verify beneficiary name |
| `ACCOUNT_FROZEN` | Account on hold | Contact support |
| `COMPLIANCE_HOLD` | Payment flagged | Wait for review |
| `INVALID_ROUTING` | Bad routing number | Verify routing info |
| `INVALID_IBAN` | Bad IBAN | Verify and correct |

### Customer Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `KYC_NOT_APPROVED` | Customer not verified | Complete KYC |
| `KYC_REJECTED` | KYC failed | Retry KYC or contact support |
| `CUSTOMER_RESTRICTED` | Account restricted | Contact compliance |
| `MISSING_INFORMATION` | Required field missing | Complete customer profile |

## Error Response Format

```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable description",
  "details": [
    {
      "field": "fieldName",
      "message": "Specific error for this field"
    }
  ],
  "requestId": "req_abc123"
}
```

## Retry Strategy

### Exponential Backoff

```python
import time

def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except GridApiError as e:
            if e.status_code in [500, 502, 503, 429]:
                wait_time = 2 ** attempt  # 1, 2, 4, 8, 16 seconds
                time.sleep(wait_time)
            else:
                raise  # Don't retry client errors
    raise Exception("Max retries exceeded")
```

### Retryable vs Non-Retryable

**Retryable (with backoff):**
- 500, 502, 503 server errors
- 429 rate limiting
- Network timeouts

**Non-Retryable:**
- 400 bad request
- 401 unauthorized
- 403 forbidden
- 404 not found
- 409 conflict
- 422 validation errors

## User-Facing Error Messages

Don't expose internal error codes to users. Map to friendly messages:

| Internal Error | User Message |
|---------------|--------------|
| INSUFFICIENT_FUNDS | "Your account balance is too low. Please add funds." |
| QUOTE_EXPIRED | "The exchange rate has expired. Please get a new quote." |
| BANK_REJECTED | "The bank rejected this transfer. Please verify account details." |
| KYC_NOT_APPROVED | "Please complete identity verification to continue." |
| COMPLIANCE_HOLD | "This payment is under review. We'll notify you when complete." |

## Logging

Log all errors with context:

```javascript
logger.error('Payment failed', {
  error: error.code,
  message: error.message,
  customerId: customerId,
  quoteId: quoteId,
  requestId: error.requestId,
  timestamp: new Date().toISOString()
});
```

## Monitoring

Track error rates:

| Metric | Alert Threshold |
|--------|----------------|
| Error rate | > 5% |
| 5xx rate | > 1% |
| Timeout rate | > 2% |
| KYC failure rate | > 20% |

## Best Practices

1. **Use idempotency keys** - Prevent duplicate operations
2. **Implement retries** - With exponential backoff
3. **Log everything** - Full context for debugging
4. **Don't leak internals** - Friendly user messages
5. **Monitor errors** - Alert on high error rates
6. **Handle all cases** - Every error needs a handler
