# External Accounts API

## Add a New External Account

```bash
POST /customers/{customerId}/external-accounts
```

## List Customer External Accounts

```bash
GET /customers/{customerId}/external-accounts
```

## Get Customer External Account by ID

```bash
GET /customers/{customerId}/external-accounts/{id}
```

## Delete Customer External Account by ID

```bash
DELETE /customers/{customerId}/external-accounts/{id}
```

## Add a New Platform External Account

```bash
POST /platform/external-accounts
```

## List Platform External Accounts

```bash
GET /platform/external-accounts
```

## External Account Types

| Type | Required Fields |
|------|----------------|
| ACH | accountNumber, routingNumber |
| SEPA | iban |
| SPEI | clabe |
| PIX | pixKey, pixKeyType |
| UPI | upiId |
| LIGHTNING_SPARK | address |
| UMA | umaAddress |
