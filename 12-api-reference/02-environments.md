# Environments

| Environment | Base URL |
|-------------|----------|
| Sandbox | `https://api.lightspark.com/grid/2025-10-13` |
| Production | `https://api.lightspark.com/grid/2025-10-13` |

## Rate Limiting

| Environment | Limit |
|-------------|-------|
| Sandbox | 100 requests/minute |
| Production | 1000 requests/minute |

## Headers

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Request limit |
| `X-RateLimit-Remaining` | Remaining requests |
| `X-RateLimit-Reset` | Reset timestamp |
