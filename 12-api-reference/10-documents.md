# Documents API

## Upload a Document

```bash
POST /documents
```

**Content-Type:** `multipart/form-data`

```bash
curl -X POST "$GRID_BASE_URL/documents" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -F "file=@document.pdf" \
  -F "documentHolderId=Customer:..." \
  -F "documentHolderType=CUSTOMER" \
  -F "documentType=GOVERNMENT_ID"
```

## Get a Document by ID

```bash
GET /documents/{id}
```

## List Documents

```bash
GET /documents?documentHolderId=Customer:...
```

## Replace a Document

```bash
POST /documents/{id}/replace
```

## Delete a Document

```bash
DELETE /documents/{id}
```

## Document Types

| Type | Description |
|------|-------------|
| GOVERNMENT_ID | Passport, driver's license, national ID |
| PROOF_OF_ADDRESS | Utility bill, bank statement |
| COMPANY_FORMATION | Certificate of incorporation |
| OWNERSHIP_CHART | Organization chart |
| TAX_ID | Tax identification documents |
