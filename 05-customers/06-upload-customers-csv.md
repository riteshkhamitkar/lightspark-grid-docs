# Upload Customers via CSV File

Upload a CSV file containing customer information for bulk creation.

## API Endpoint

```bash
POST /customers/bulk-upload
```

## Request

```bash
curl --request POST \
  --url https://api.lightspark.com/grid/2025-10-13/customers/bulk-upload \
  --header 'Authorization: Basic <encoded-value>' \
  --form 'file=@/path/to/customers.csv'
```

## CSV Format

### Required Columns

| Column | Description | Example |
|--------|-------------|---------|
| `customerType` | `INDIVIDUAL` or `BUSINESS` | INDIVIDUAL |
| `platformCustomerId` | Your unique identifier | cust_001 |
| `fullName` | Full legal name | Jane Smith |
| `email` | Email address | jane@example.com |
| `region` | ISO country code | US |

### Individual Columns

| Column | Description | Example |
|--------|-------------|---------|
| `birthDate` | Date of birth | 1990-01-15 |
| `nationality` | Nationality country code | US |
| `address.line1` | Street address | 123 Main St |
| `address.city` | City | San Francisco |
| `address.state` | State | CA |
| `address.postalCode` | ZIP code | 94105 |
| `address.country` | Country | US |

### Business Columns

| Column | Description | Example |
|--------|-------------|---------|
| `businessInfo.legalName` | Legal entity name | Acme Corp |
| `businessInfo.registrationNumber` | Tax/registration ID | DE123456 |
| `businessInfo.entityType` | Entity type | LLC |

## CSV Example

```csv
customerType,platformCustomerId,fullName,email,region,birthDate,nationality,address.line1,address.city,address.state,address.postalCode,address.country
INDIVIDUAL,cust_001,Jane Smith,jane@example.com,US,1990-01-15,US,123 Main St,San Francisco,CA,94105,US
INDIVIDUAL,cust_002,John Doe,john@example.com,US,1985-05-20,US,456 Oak Ave,New York,NY,10001,US
BUSINESS,biz_001,Acme Corporation,finance@acme.com,US,,,789 Corp Blvd,Los Angeles,CA,90001,US
```

## Response

```json
{
  "jobId": "BulkUpload:019542f5-b3e7-1d02-0000-000000000001",
  "status": "PROCESSING",
  "totalRows": 100,
  "processedRows": 0,
  "createdAt": "2025-01-15T10:00:00Z"
}
```

## Check Upload Status

```bash
GET /customers/bulk-upload/{jobId}
```

```bash
curl --request GET \
  --url https://api.lightspark.com/grid/2025-10-13/customers/bulk-upload/BulkUpload:... \
  --header 'Authorization: Basic <encoded-value>'
```

### Response

```json
{
  "jobId": "BulkUpload:019542f5-b3e7-1d02-0000-000000000001",
  "status": "COMPLETED",
  "totalRows": 100,
  "processedRows": 100,
  "successfulRows": 98,
  "failedRows": 2,
  "errors": [
    {
      "row": 45,
      "error": "Invalid email format",
      "data": { ... }
    },
    {
      "row": 67,
      "error": "Missing required field: birthDate",
      "data": { ... }
    }
  ],
  "createdAt": "2025-01-15T10:00:00Z",
  "completedAt": "2025-01-15T10:05:00Z"
}
```

## Status Values

| Status | Description |
|--------|-------------|
| `PENDING` | Upload received, waiting to process |
| `PROCESSING` | Currently processing rows |
| `COMPLETED` | All rows processed |
| `FAILED` | Upload failed entirely |

## Webhook Event

`bulk_upload.status_change` - Sent when upload completes:

```json
{
  "eventType": "bulk_upload.status_change",
  "data": {
    "jobId": "BulkUpload:...",
    "status": "COMPLETED",
    "totalRows": 100,
    "successfulRows": 98,
    "failedRows": 2
  }
}
```

## Best Practices

1. **Validate CSV first** - Check format before uploading
2. **Limit file size** - Maximum 10MB (approximately 10,000 rows)
3. **Use UTF-8 encoding** - Avoid character encoding issues
4. **Include headers** - First row must be column headers
5. **Test with small batch** - Upload 10-50 rows first
6. **Handle errors** - Review failed rows and retry
7. **Monitor webhook** - Wait for completion notification
