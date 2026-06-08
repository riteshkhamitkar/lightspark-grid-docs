# Exporting a Wallet

Let a customer take the seed of their Global Account off Grid.

## Why Export?

Self-custody means customers truly own their funds. Exporting the wallet seed allows customers to:

- Access funds outside of your platform
- Use other Spark-compatible wallets
- Maintain access if your platform is unavailable
- Have full control over their private keys

## Export Process

### 1. Authenticate Customer

Customer must authenticate with their credential (passkey, OTP, etc.).

### 2. Request Export

```bash
curl -X POST "$GRID_BASE_URL/internal-accounts/InternalAccount:ga-abc123/export" \
  -u "$GRID_CLIENT_ID:$GRID_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "credentialId": "Credential:passkey-1",
    "signature": "base64_signature..."
  }'
```

### 3. Receive Encrypted Seed

```json
{
  "encryptedSeed": "base64_encrypted_seed...",
  "encryptionMethod": "HPKE",
  "publicKey": "base64_public_key..."
}
```

### 4. Decrypt Seed on Device

The seed is encrypted with the customer's public key. Only their private key can decrypt it.

```javascript
const seed = await hpkeDecrypt(
  devicePrivateKey,
  base64ToBuffer(encryptedSeed)
);
```

## Seed Format

The exported seed follows the BIP-39 standard (mnemonic phrase) or BIP-32 (extended private key), depending on the wallet implementation.

## Security Considerations

1. **Seed is sensitive** - Never transmit unencrypted seeds
2. **Customer responsibility** - Customer must secure their seed
3. **Irreversible** - Cannot be undone; seed is permanently exposed
4. **Platform notification** - Consider notifying platform of exports

## Importing to Other Wallets

The exported seed can be imported into:
- Spark wallet (spark.money)
- Other Lightning-compatible wallets
- Bitcoin wallets supporting the same derivation path

## Best Practices

1. **Clear warnings** - Explain the responsibility of holding a seed
2. **Secure delivery** - Ensure encrypted transport
3. **Verify identity** - Strong authentication before export
4. **Audit logging** - Log all export attempts
5. **Customer education** - Explain how to secure the seed
