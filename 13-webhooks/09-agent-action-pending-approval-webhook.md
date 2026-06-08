# Agent Action Pending Approval Webhook

Fired when an agent submits an action that requires platform approval before Grid will execute it.

## Event Type

```
agent.action_pending_approval
```

## Payload

```json
{
  "eventType": "agent.action_pending_approval",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "actionId": "AgentAction:...",
    "agentId": "Agent:...",
    "customerId": "Customer:...",
    "actionType": "EXECUTE_QUOTE",
    "amount": "100.00",
    "currency": "USD",
    "description": "Payment to vendor"
  }
}
```

## Handling

1. Send push notification to customer
2. Show approval UI in app
3. Allow approve/reject
4. Call API to confirm decision
