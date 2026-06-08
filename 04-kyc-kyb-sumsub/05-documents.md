# Documents

Upload and manage verification documents for customers and beneficial owners.

## Supported Document Types

| Document Type | Description | Used For |
|--------------|-------------|----------|
| `GOVERNMENT_ID` | Passport, driver's license, national ID | KYC/KYB identity |
| `PROOF_OF_ADDRESS` | Utility bill, bank statement, lease | Address verification |
| `COMPANY_FORMATION` | Certificate of incorporation, articles | KYB business verification |
| `OWNERSHIP_CHART` | Organization chart, shareholder agreement | KYB ownership proof |
| `TAX_ID` | Tax identification documents | KYB tax verification |

## API Endpoints

### Upload a Document

```bash
POST /documents
```

**Content-Type:** `multipart/form-data`

```bash
curl --request POST \
  --url https://api.lightspark.com/grid/2025-10-13/documents \
  --header 'Authorization: Basic <encoded-value>' \
  --form 'file=@/path/to/document.pdf' \
  --form 'documentHolderId=Customer:...' \
  --form 'documentHolderType=CUSTOMER' \
  --form 'documentType=GOVERNMENT_ID'
```

### Request Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | file | Yes | The document file (JPG, PNG, PDF) |
| `documentHolderId` | string | Yes | Customer or BeneficialOwner ID |
| `documentHolderType` | string | Yes | `CUSTOMER` or `BENEFICIAL_OWNER` |
| `documentType` | string | Yes | Document type code |
| `description` | string | No | Optional description |

### File Requirements

| Requirement | Specification |
|-------------|--------------|
| Formats | JPG, PNG, PDF |
| Max size | 10 MB |
| Min resolution | 300 DPI for IDs |
| Color | Color photos required |
| Expired docs | Not accepted for IDs |

### Get a Document

```bash
GET /documents/{id}
```

```bash
curl --request GET \
  --url https://api.lightspark.com/grid/2025-10-13/documents/Document:... \
  --header 'Authorization: Basic <encoded-value>'
```

### List Documents

```bash
GET /documents?documentHolderId=Customer:...
```

| Query Parameter | Description |
|----------------|-------------|
| `documentHolderId` | Filter by holder |
| `documentType` | Filter by type |
| `status` | Filter by status |

### Replace a Document

```bash
POST /documents/{id}/replace
```

Use when a document was rejected and needs to be re-uploaded:

```bash
curl --request POST \
  --url https://api.lightspark.com/grid/2025-10-13/documents/Document:.../replace \
  --header 'Authorization: Basic <encoded-value>' \
  --form 'file=@/path/to/new_document.pdf' \
  --form 'documentType=GOVERNMENT_ID'
```

### Delete a Document

```bash
DELETE /documents/{id}
```

Note: Documents already submitted for verification may not be deletable.

## Document Statuses

| Status | Description |
|--------|-------------|
| `PENDING` | Uploaded, not yet reviewed |
| `UNDER_REVIEW` | Being reviewed |
| `APPROVED` | Document accepted |
| `REJECTED` | Document rejected (see reason) |

## Common Rejection Reasons

| Reason | Solution |
|--------|----------|
| Blurry image | Retake with better focus |
| Glare on ID | Avoid direct light, use indirect lighting |
| Cropped corners | Show all four corners of ID |
| Expired document | Use current, valid ID |
| Wrong document type | Upload correct document type |
| Name mismatch | Ensure name matches customer record |

## Upload Flow

### For Individual KYC

1. Upload government ID
   ```
   POST /documents
   - documentType: GOVERNMENT_ID
   - documentHolderType: CUSTOMER
   ```

2. Upload proof of address (if required)
   ```
   POST /documents
   - documentType: PROOF_OF_ADDRESS
   - documentHolderType: CUSTOMER
   ```

### For Business KYB

1. Upload company formation documents
   ```
   POST /documents
   - documentType: COMPANY_FORMATION
   - documentHolderType: CUSTOMER
   ```

2. Upload ID for each beneficial owner
   ```
   POST /documents
   - documentType: GOVERNMENT_ID
   - documentHolderType: BENEFICIAL_OWNER
   - documentHolderId: BeneficialOwner:...
   ```

3. Upload proof of address
   ```
   POST /documents
   - documentType: PROOF_OF_ADDRESS
   - documentHolderType: CUSTOMER
   ```

4. Upload ownership documents
   ```
   POST /documents
   - documentType: OWNERSHIP_CHART
   - documentHolderType: CUSTOMER
   ```

## Best Practices

1. **Upload before submitting verification** - Have all docs ready
2. **Check file size** - Keep under 10MB
3. **Use clear scans** - 300+ DPI for IDs
4. **Show all corners** - Full document visible
5. **Valid documents only** - Check expiration dates
6. **Match names** - Document name must match record
