# Transactions API

## Get Transaction by ID

```bash
GET /transactions/{id}
```

## List Transactions

```bash
GET /transactions
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| `customerId` | Filter by customer |
| `platformCustomerId` | Filter by your customer ID |
| `status` | `PENDING`, `IN_FLIGHT`, `COMPLETED`, `FAILED` |
| `currency` | Filter by currency |
| `fromDate` | Start date |
| `toDate` | End date |
| `type` | `TRANSFER_IN`, `TRANSFER_OUT`, `CONVERSION` |
| `page` | Page number |
| `limit` | Results per page |

## Approve a Pending Incoming Payment

```bash
POST /transactions/{id}/approve
```

For payments that require manual approval.

## Reject a Pending Incoming Payment

```bash
POST /transactions/{id}/reject
```

## Confirm Receipt Delivery

```bash
POST /transactions/{id}/confirm-receipt
```
