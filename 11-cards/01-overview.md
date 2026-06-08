# Cards

Issue virtual debit cards, decision authorizations in real time, and reconcile transactions against internal accounts.

With Grid Cards, you can issue virtual debit cards backed by an internal account, decision authorizations in real time against that account's balance, and reconcile every pull, clearing, and refund against the same ledger you already use for payouts.

## Features

### Real-Time Decisioning
Authorizations are checked against the bound internal account at auth time, so the funding source and the card share a single source of truth.

### No PAN in Your Stack
The full PAN and CVV are rendered directly to the cardholder through the issuer's iframe. Card credentials never cross your servers.

## Card Lifecycle

A card moves through five states:

| State | Meaning |
|-------|---------|
| `PENDING_KYC` | Cardholder has not finished KYC; the card cannot transact yet |
| `PENDING_ISSUE` | Card has been requested and is being provisioned with the issuer |
| `ACTIVE` | Card is live and can authorize transactions |
| `FROZEN` | Card is temporarily disabled; new authorizations are declined; in-flight settlements continue |
| `CLOSED` | Card is permanently closed; terminal, irreversible |

Every transition emits a `CARD.STATE_CHANGE` webhook so you can mirror state changes into your application.

## What's Covered

- **Quickstart** - Issue your first card, simulate an authorization, and watch it reconcile against an internal account
- **Card Management** - Issue cards, bind funding sources, freeze, unfreeze, and close
- **Transactions & Reconciliation** - The relationship between authorizations, pulls, clearings, and refunds
- **Sandbox Testing** - Drive deterministic outcomes with magic-value suffix tables
