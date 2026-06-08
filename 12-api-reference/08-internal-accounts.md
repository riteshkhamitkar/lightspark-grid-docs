# Internal Accounts API

## List Customer Internal Accounts

```bash
GET /internal-accounts
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| `customerId` | Filter by customer |
| `currency` | Filter by currency |
| `accountOwnerType` | `CUSTOMER` or `PLATFORM` |

## List Platform Internal Accounts

```bash
GET /internal-accounts?accountOwnerType=PLATFORM
```

## Get Internal Account by ID

```bash
GET /internal-accounts/{id}
```

## Update Internal Account

```bash
PATCH /internal-accounts/{id}
```

## Export Internal Account Wallet Credentials (Global Accounts)

```bash
POST /internal-accounts/{id}/export
```

### Request Body

```json
{
  "credentialId": "Credential:...",
  "clientPublicKey": "base64_public_key..."
}
```

### Response

```json
{
  "encryptedSeed": "base64_encrypted_seed...",
  "encryptionMethod": "HPKE"
}
```
