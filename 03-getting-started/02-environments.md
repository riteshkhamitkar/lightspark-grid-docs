# Environments

Grid provides two environments for development and production.

## Sandbox Environment

**Base URL:** `https://api.lightspark.com/grid/2025-10-13`

The sandbox environment is a complete testing environment that simulates all API behavior without moving real money.

### Sandbox Features
- Free to use
- No real money movement
- Instant KYC approval (with magic values)
- Simulated settlement
- Magic values for testing different scenarios
- Shared environment (don't use real customer data)

### Getting Sandbox Credentials
1. Create account at [app.lightspark.com](https://app.lightspark.com)
2. Generate sandbox API keys from the dashboard
3. Start building immediately

## Production Environment

**Base URL:** `https://api.lightspark.com/grid/2025-10-13`

The production environment handles real money and requires approval.

### Production Features
- Real money movement
- Full KYC/KYB verification
- Actual bank settlement times
- Requires Lightspark approval
- Dedicated environment

### Getting Production Access
1. Complete integration testing in sandbox
2. Contact your Lightspark representative
3. Complete compliance review
4. Receive production credentials
5. Launch with real transactions

## Environment Differences

| Feature | Sandbox | Production |
|---------|---------|------------|
| Real money | No | Yes |
| KYC verification | Magic values | Real SumSub process |
| Settlement | Simulated | Real bank times |
| Cost | Free | Per-transaction fees |
| Rate limits | Lower | Higher |
| Data persistence | May be reset | Permanent |

## Headers

### Idempotency Key
Use for safe retries:
```bash
Idempotency-Key: unique-key-123
```

### Request ID
Grid returns a request ID in every response:
```bash
X-Request-ID: req_abc123
```

### Content Type
Always include:
```bash
Content-Type: application/json
```

## API Versioning

Current version: `2025-10-13`

The version is included in the URL path. When new versions are released, you'll have time to migrate before deprecation.

## Rate Limiting

| Environment | Rate Limit |
|-------------|-----------|
| Sandbox | 100 requests/minute |
| Production | 1000 requests/minute |

Rate limit headers:
```bash
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

## Webhook URLs Per Environment

Configure separate webhook URLs for each environment:
- Sandbox webhooks: `https://sandbox.your-domain.com/webhooks/grid`
- Production webhooks: `https://your-domain.com/webhooks/grid`

## Testing Best Practices

1. **Test all flows in sandbox first**
2. **Use magic values** to simulate edge cases
3. **Test error handling** with failure scenarios
4. **Verify webhook handling** end-to-end
5. **Load test** with realistic volumes
6. **Document your integration** for your team
