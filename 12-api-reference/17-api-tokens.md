# API Tokens API

## Create a New API Token

```bash
POST /api-tokens
```

```bash
curl -X POST "$GRID_BASE_URL/api-tokens" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Production API Key",
    "environment": "production"
  }'
```

## List Tokens

```bash
GET /api-tokens
```

## Get API Token by ID

```bash
GET /api-tokens/{id}
```

## Delete API Token by ID

```bash
DELETE /api-tokens/{id}
```

## Token Fields

| Field | Description |
|-------|-------------|
| `id` | Token ID |
| `name` | Human-readable name |
| `environment` | `sandbox` or `production` |
| `createdAt` | Creation timestamp |
| `lastUsedAt` | Last usage timestamp |
