# Agent Operations API

Operations performed by authenticated agents.

## Get Current Agent

```bash
GET /agents/me
```

## Add an External Account

```bash
POST /agents/external-accounts
```

## List Agent External Accounts

```bash
GET /agents/external-accounts
```

## Get Agent External Account by ID

```bash
GET /agents/external-accounts/{id}
```

## Delete Agent External Account

```bash
DELETE /agents/external-accounts/{id}
```

## Create a Transfer Quote

```bash
POST /agents/quotes
```

## Execute a Quote

```bash
POST /agents/quotes/{id}/execute
```

## Get Agent Quote by ID

```bash
GET /agents/quotes/{id}
```

## Create a Transfer-In

```bash
POST /agents/transfers/in
```

## Create a Transfer-Out

```bash
POST /agents/transfers/out
```

## List Agent Transactions

```bash
GET /agents/transactions
```

## Get Agent Transaction by ID

```bash
GET /agents/transactions/{id}
```

## Get an Agent Action

```bash
GET /agents/actions/{id}
```

## List Agent's Own Actions

```bash
GET /agents/actions
```

## List Agent's Internal Accounts

```bash
GET /agents/internal-accounts
```
