# Currencies & Payment Rails

Supported currencies, countries, and payment methods.

Grid supports a wide range of fiat currencies and cryptocurrencies, with automatic routing across optimal payment rails based on currency, destination, and speed requirements.

## Supported Currencies

### Cryptocurrency (Available Globally)
- **Bitcoin (BTC)** - Via Lightning Network
- **Stablecoins** - USDC, USDB (Brale-issued)

### Fiat Currencies

| Currency | Code | Supported Rails |
|----------|------|-----------------|
| US Dollar | USD | ACH, Wire, RTP |
| Euro | EUR | SEPA, SEPA Instant |
| Brazilian Real | BRL | PIX |
| Mexican Peso | MXN | CLABE |
| Indian Rupee | INR | UPI |
| Philippine Peso | PHP | InstaPay, PESONet |
| British Pound | GBP | FPS, SEPA |
| Colombian Peso | COP | Bank Transfer |
| and 50+ more | | |

## Countries & Payment Rails

### Full Platform Access (US & Europe)
Onboard as a platform with complete access to APIs, hosted KYC/KYB, dashboard, and business integrations.

### Local Payment Rails (60+ Countries)

| Country | ISO | Payment Rails |
|---------|-----|---------------|
| Austria | AT | SEPA, SEPA Instant |
| Bangladesh | BD | Bank Transfer |
| Belgium | BE | SEPA, SEPA Instant |
| Benin | BJ | Bank Transfer |
| Brazil | BR | PIX |
| Bulgaria | BG | SEPA, SEPA Instant |
| Cameroon | CM | Bank Transfer |
| China | CN | Bank Transfer |
| Colombia | CO | Bank Transfer |
| Croatia | HR | SEPA, SEPA Instant |
| Cyprus | CY | SEPA, SEPA Instant |
| Czech Republic | CZ | SEPA, SEPA Instant |
| Denmark | DK | SEPA, SEPA Instant |
| Estonia | EE | SEPA, SEPA Instant |
| Finland | FI | SEPA, SEPA Instant |
| France | FR | SEPA, SEPA Instant |
| Germany | DE | SEPA, SEPA Instant |
| Ghana | GH | Bank Transfer |
| Greece | GR | SEPA, SEPA Instant |
| Hong Kong | HK | FPS |
| Hungary | HU | SEPA, SEPA Instant |
| India | IN | UPI |
| Indonesia | ID | Bank Transfer |
| Ireland | IE | SEPA, SEPA Instant |
| Italy | IT | SEPA, SEPA Instant |
| Ivory Coast | CI | Bank Transfer |
| Japan | JP | Bank Transfer |
| Kenya | KE | Bank Transfer |
| Latvia | LV | SEPA, SEPA Instant |
| Lithuania | LT | SEPA, SEPA Instant |
| Luxembourg | LU | SEPA, SEPA Instant |
| Malaysia | MY | Bank Transfer |
| Malta | MT | SEPA, SEPA Instant |
| Mexico | MX | CLABE |
| Netherlands | NL | SEPA, SEPA Instant |
| Nigeria | NG | Bank Transfer |
| Norway | NO | SEPA, SEPA Instant |
| Pakistan | PK | Bank Transfer |
| Peru | PE | Bank Transfer |
| Philippines | PH | InstaPay, PESONet |
| Poland | PL | SEPA, SEPA Instant |
| Portugal | PT | SEPA, SEPA Instant |
| Romania | RO | SEPA, SEPA Instant |
| Senegal | SN | Bank Transfer |
| Singapore | SG | Bank Transfer |
| Slovakia | SK | SEPA, SEPA Instant |
| Slovenia | SI | SEPA, SEPA Instant |
| South Africa | ZA | Bank Transfer |
| Spain | ES | SEPA, SEPA Instant |
| Sri Lanka | LK | Bank Transfer |
| Sweden | SE | SEPA, SEPA Instant |
| Switzerland | CH | SEPA, SEPA Instant |
| Thailand | TH | Bank Transfer |
| Turkey | TR | Bank Transfer |
| UAE | AE | Bank Transfer |
| Uganda | UG | Bank Transfer |
| United Kingdom | GB | FPS, SEPA |
| United States | US | ACH, Wire, RTP |
| Vietnam | VN | Bank Transfer |

## Currency Conversion

### Exchange Rates

- Rates are cached for approximately 5 minutes
- Quotes lock in rates for 1-15 minutes
- Rates include fees specific to your platform
- Get current rates via: `GET /exchange-rates`

### Example Conversion Flow

**USD to EUR:**
1. Create quote with USD source, EUR destination
2. Quote returns locked rate (e.g., 1 USD = 0.92 EUR)
3. Execute quote - Grid handles FX conversion
4. Recipient receives EUR via SEPA

**USD to BTC:**
1. Create quote with USD internal account, BTC external wallet
2. Quote returns locked rate (e.g., 1 USD = 0.000015 BTC)
3. Execute quote - Grid converts USD to BTC
4. BTC sent via Lightning Network

## Payment Rail Characteristics

### Instant Rails (Seconds)
- Lightning Network
- UMA
- PIX (Brazil)
- SEPA Instant (EU)
- UPI (India)
- FPS (UK/Hong Kong)
- RTP (US)

### Fast Rails (Minutes to Hours)
- SEPA (EU) - Same day
- ACH (US) - 1-3 business days
- InstaPay (Philippines) - Minutes

### Standard Rails (Days)
- Wire (US) - Same day to 1 day
- Bank Transfer (Various) - 1-5 business days

## Rail Selection

Grid automatically selects the optimal rail based on:
1. **Destination country** - Available rails in that country
2. **Speed requirements** - Instant vs standard settlement
3. **Cost optimization** - Lowest fee route
4. **Currency** - Native rail for the currency
5. **Account type** - Which rails the external account supports

You can see available rails for a country using the discovery API:
```bash
GET /discoveries?country=PH&currency=PHP
```

## Coming Soon

Grid continuously adds new currencies and payment rails. Check the documentation for the latest additions.
