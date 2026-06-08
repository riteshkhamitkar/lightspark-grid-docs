# Card Terminology

Concepts and resources specific to the Cards API.

## Key Terms

### Card
A virtual debit card issued to a cardholder. Linked to one or more internal accounts as funding sources.

### Cardholder
A customer who has been issued a card. Must have KYC status `APPROVED`.

### Funding Source
An internal account linked to a card. Used to fund card transactions.

### Authorization
A real-time check when a card is used for a purchase. Checks if the funding source has sufficient balance.

### Clearing (Settlement)
The actual transfer of funds from the funding source to the merchant after authorization.

### Pull
A debit transaction against the funding source.

### Refund
A return of funds to the funding source.

### PAN
Primary Account Number - the card number. Never touches your servers.

### CVV
Card Verification Value - the 3-digit security code.

### Card Transaction
A transaction record for card activity (authorization, clearing, refund).

## Abbreviations

| Abbreviation | Meaning |
|-------------|---------|
| PAN | Primary Account Number |
| CVV | Card Verification Value |
| KYC | Know Your Customer |
| 3DS | 3D Secure |
| PIN | Personal Identification Number |
