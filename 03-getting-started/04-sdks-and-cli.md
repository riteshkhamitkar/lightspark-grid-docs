# SDKs & CLI

Official client libraries and command-line interface for the Grid API.

## Official SDKs

### JavaScript/TypeScript

```bash
npm install @lightsparkdev/grid-sdk
```

```javascript
import { GridClient } from '@lightsparkdev/grid-sdk';

const grid = new GridClient({
  baseUrl: 'https://api.lightspark.com/grid/2025-10-13',
  clientId: process.env.GRID_CLIENT_ID,
  clientSecret: process.env.GRID_CLIENT_SECRET,
});

// Create a customer
const customer = await grid.customers.create({
  customerType: 'INDIVIDUAL',
  fullName: 'Jane Doe',
  email: 'jane@example.com',
  ...
});
```

### Python

```bash
pip install lightspark-grid
```

```python
import lightspark_grid

grid = lightspark_grid.GridClient(
    base_url='https://api.lightspark.com/grid/2025-10-13',
    client_id=os.environ['GRID_CLIENT_ID'],
    client_secret=os.environ['GRID_CLIENT_SECRET']
)

# Create a customer
customer = grid.customers.create(
    customer_type='INDIVIDUAL',
    full_name='Jane Doe',
    email='jane@example.com',
    ...
)
```

## CLI Tool

### Installation

```bash
npm install -g @lightsparkdev/grid-cli
```

### Authentication

```bash
grid auth login
# Enter your Client ID and Client Secret
```

### Common Commands

```bash
# List customers
grid customers list

# Get customer details
grid customers get Customer:...

# Create customer
grid customers create --type INDIVIDUAL --name "Jane Doe" ...

# List internal accounts
grid internal-accounts list

# Check balance
grid internal-accounts get InternalAccount:...

# Create quote
grid quotes create --source ... --destination ... --amount ...

# Execute quote
grid quotes execute Quote:...

# List transactions
grid transactions list
```

## Environment Setup

### Required Environment Variables

```bash
export GRID_CLIENT_ID="your_client_id"
export GRID_CLIENT_SECRET="your_client_secret"
export GRID_BASE_URL="https://api.lightspark.com/grid/2025-10-13"
```

### Optional Environment Variables

```bash
export GRID_WEBHOOK_SECRET="your_webhook_secret"
export GRID_LOG_LEVEL="debug"
```

## Type Definitions

### TypeScript Types

```typescript
import { Customer, InternalAccount, Quote, Transaction } from '@lightsparkdev/grid-sdk';

// All types are exported for type-safe development
const customer: Customer = await grid.customers.get(id);
```

## Error Handling

SDKs throw typed errors:

```typescript
try {
  await grid.quotes.execute(quoteId);
} catch (error) {
  if (error instanceof GridApiError) {
    console.log(error.statusCode); // HTTP status
    console.log(error.code);       // Error code
    console.log(error.message);    // Error message
  }
}
```

## Retry Logic

SDKs include automatic retry with exponential backoff for:
- 5xx server errors
- Network timeouts
- Rate limit responses (429)

## Postman Collection

Grid provides an official Postman collection:
- Import from: https://docs.lightspark.com/postman-collection
- Includes all endpoints with examples
- Pre-configured environments for sandbox and production

## Building Custom SDKs

If you need a custom SDK, the OpenAPI spec is available at:
https://docs.lightspark.com/openapi.yaml

You can generate SDKs using tools like:
- OpenAPI Generator
- Swagger Codegen
- Kiota (Microsoft)
