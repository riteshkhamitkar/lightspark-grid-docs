# Bulk Upload Status Webhook

Webhook that is called when a bulk customer upload job completes or fails.

## Event Type

```
bulk_upload.status_change
```

## Payload

```json
{
  "eventType": "bulk_upload.status_change",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "jobId": "BulkUpload:...",
    "status": "COMPLETED",
    "totalRows": 100,
    "successfulRows": 98,
    "failedRows": 2,
    "errors": [
      { "row": 45, "error": "Invalid email" }
    ]
  }
}
```

## Handling

1. Notify admin of completion
2. Log failed rows for review
3. Trigger follow-up for failures
4. Update import dashboard
