# External Accounts for Rewards

Add and manage external funding sources and wallets as payment destinations for rewards.

Same as [Ramps external accounts](../08-ramps/07-external-accounts.md).

For rewards, external accounts are recipient BTC wallets:

```bash
curl -X POST "$GRID_BASE_URL/customers/Customer:.../external-accounts" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "accountType": "LIGHTNING_SPARK",
    "address": "$user@wallet.com",
    "currency": "BTC",
    "accountOwnerName": "Recipient Name"
  }'
```

## Collecting Wallet Addresses

Methods to collect recipient wallet addresses:

1. **User input** - Ask users to provide their address
2. **Deep link** - Link to Spark wallet with pre-filled details
3. **QR code** - User scans QR from their wallet
4. **Connect wallet** - Web3-style wallet connection

## Validation

Always validate Bitcoin addresses before creating external accounts:

```javascript
function isValidLightningAddress(address) {
  return /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(address);
}

function isValidBtcAddress(address) {
  // Check for legacy, segwit, or bech32 formats
  return /^((bc1|[13])[a-zA-HJ-NP-Z0-9]{25,62})$/.test(address);
}
```

## Best Practices

1. **Validate addresses** - Check format before sending
2. **Educate users** - Explain what address to provide
3. **Test first** - Send small test amount
4. **Confirm receipt** - Verify wallet received funds
