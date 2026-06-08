# Agent Management API

Experimental native Grid functionality for connecting AI agents.

## Create an Agent

```bash
POST /agents
```

```bash
curl -X POST "$GRID_BASE_URL/agents" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Payment Agent",
    "policy": {
      "permissions": ["CREATE_QUOTES", "EXECUTE_QUOTES"],
      "spendingLimit": "1000.00",
      "currency": "USD",
      "approvalRequired": true
    }
  }'
```

## List Agents

```bash
GET /agents
```

## Get Agent by ID

```bash
GET /agents/{id}
```

## Update Agent

```bash
PATCH /agents/{id}
```

## Update Agent Policy

```bash
PATCH /agents/{id}/policy
```

## Delete Agent

```bash
DELETE /agents/{id}
```

## Regenerate a Device Code

```bash
POST /agents/{id}/device-code
```

## Redeem Device Code

```bash
POST /auth/device-code/redeem
```

## Get Device Code Status

```bash
GET /auth/device-code/{code}/status
```

## List Agent Transaction Approval Requests

```bash
GET /agents/{id}/approval-requests
```

## Approve an Agent Action

```bash
POST /agents/{id}/actions/{actionId}/approve
```

## Reject an Agent Action

```bash
POST /agents/{id}/actions/{actionId}/reject
```
