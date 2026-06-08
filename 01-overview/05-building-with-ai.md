# Building with AI

Use AI coding assistants like Claude Code, Cursor, and Codex to explore, build, and debug with the Grid API.

## Tips for AI-Assisted Development

### 1. Reference the Documentation

Point your AI assistant to the official documentation:
- **LLMs.txt:** https://docs.lightspark.com/llms.txt (complete index)
- **OpenAPI Spec:** https://docs.lightspark.com/openapi.yaml

### 2. Use the Flow Builder

The [Flow Builder](https://docs.lightspark.com/flow-builder) helps you design payment flows and get the exact API calls. Share the generated cURL with your AI assistant to generate code in your preferred language.

### 3. Start with the Quickstarts

Each product section has a quickstart guide with complete working examples. Use these as a foundation.

### 4. Use Sandbox Magic Values

Test different scenarios using sandbox magic values. Tell your AI assistant which scenario you're testing.

### 5. Common Patterns

Here are common integration patterns to discuss with your AI:

#### Pattern 1: Prefunded Payout
```
1. Create customer with KYC
2. Fund internal account (ACH push)
3. Add external account (beneficiary)
4. Create quote (internal -> external)
5. Execute quote
```

#### Pattern 2: JIT Funded Payout
```
1. Create customer with KYC
2. Add external account (beneficiary)
3. Create quote (JIT -> external)
4. Follow payment instructions to fund
5. Grid executes automatically
```

#### Pattern 3: Global Account Withdrawal
```
1. Create customer (auto-provisions Global Account)
2. Complete KYC
3. Fund Global Account
4. Register passkey/authentication
5. Add external bank account
6. Create withdrawal quote
7. Sign with session key
8. Execute with signature
```

### 6. Debugging with AI

When something goes wrong:
1. Share the API response (status code, error message)
2. Share relevant webhook payloads
3. Reference the [Error Handling](../02-core-concepts/) guides
4. Check [Sandbox Magic Values](../03-getting-started/03-sandbox-testing.md) if testing

### 7. Code Generation Prompts

Effective prompts for AI assistants:

**"Generate a function to send a cross-border payout from USD to MXN using Grid API"**

**"Create a webhook handler for Grid payment status changes in Node.js"**

**"Write Python code to onboard a business customer with KYB verification"**

**"Generate a complete Global Account authentication flow with passkey registration"**

### 8. SDK Integration

```bash
# Install Grid SDK
npm install @lightsparkdev/grid-sdk

# Or using environment variables
export GRID_CLIENT_ID="your_client_id"
export GRID_CLIENT_SECRET="your_client_secret"
```

### 9. Testing Patterns

```javascript
// Example: Test KYC flow with magic values
const customer = await grid.customers.create({
  customerType: "INDIVIDUAL",
  fullName: "Test User 001", // 001 suffix = KYC APPROVED
  // ...
});
```

### 10. Security Best Practices

- Never hardcode API credentials
- Use webhook signature verification
- Implement proper idempotency keys
- Store customer data securely
- Follow OAuth 2.0 for Global Account auth
