# Configuring Customers for Payouts

Creating and managing customers for Payouts & B2B payments.

## Customer Types

### Individual
For personal payouts, freelancers, contractors.

### Business
For B2B payments, vendor payouts, AP automation.

## Creating a Customer

```bash
POST /customers
```

### Individual Example

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "INDIVIDUAL",
    "platformCustomerId": "payout-001",
    "region": "US",
    "fullName": "Jane Doe",
    "birthDate": "1990-01-15",
    "nationality": "US",
    "email": "jane@example.com",
    "address": {
      "line1": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postalCode": "94105",
      "country": "US"
    }
  }'
```

### Business Example

```bash
curl -X POST "$GRID_BASE_URL/customers" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "customerType": "BUSINESS",
    "platformCustomerId": "biz-001",
    "region": "US",
    "businessInfo": {
      "legalName": "Acme Corp",
      "registrationNumber": "DE123456789",
      "entityType": "CORPORATION"
    },
    "email": "finance@acme.com",
    "address": {
      "line1": "456 Corp Blvd",
      "city": "New York",
      "state": "NY",
      "postalCode": "10001",
      "country": "US"
    }
  }'
```

## KYC/KYB Flow

### For Individuals

1. Create customer
2. Generate KYC link
3. Customer completes verification
4. Receive webhook on completion

### For Businesses

1. Create business customer
2. Add beneficial owners
3. Upload company documents
4. Submit for verification
5. Receive webhook on completion

## Customer Record Keeping

Store in your database:
- Grid customer ID
- Your platform customer ID
- KYC status
- Associated internal accounts
- Payout history

## Best Practices

1. **Unique platformCustomerId** - Use your internal ID
2. **Complete information** - Fill all required fields
3. **Start KYC early** - Don't wait until first payout
4. **Track status** - Monitor KYC progress
5. **Handle rejections** - Have process for failed KYC
