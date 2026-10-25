# Module 04 — Currency & Exchange Rates

**Status:** Draft  
**Phase:** Foundation  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Handles all money representation, storage, and currency conversion across the system. The core principle: **money is always stored as integers in minor units** (no floats, ever). Exchange rates are versioned with effective dates so that historical reports show the rate that was in effect at the time of each transaction.

Scope: minor-unit storage conventions, exchange rate table, rate locking on transactions, multi-currency settlement, exchange gain/loss, and reporting in base currency.

---

## Money Representation

### Storage rule

All monetary columns in the database are `bigint` (unsigned where appropriate) storing the amount in the **minor unit** of the relevant currency.

| Currency | Minor unit | Decimal places | Example |
|----------|------------|----------------|---------|
| OMR | Baisa | 3 | 1.500 OMR = 1500 |
| PKR | Paisa | 2 | 150.00 PKR = 15000 |
| USD | Cent | 2 | 10.00 USD = 1000 |

> PKR uses 2 decimal places (subunit = 100: 1 rupee = 100 paisa). Amounts display as rupees in the UI; the database always stores paisa. The pack declares `currency_subunit = 100` for PKR. Code never hard-codes subunits.

### Column naming

All monetary columns use the suffix `_amount` and carry a sibling `_currency_code` column:

```
advance_amount       bigint
advance_currency_code char(3)
```

For amounts that are always in the branch's operating currency, the currency column may be omitted and the currency assumed from context — but this must be documented on the model.

### PHP money class

Use a value object (e.g. `App\ValueObjects\Money`) that wraps `amount` (int) + `currency_code`. It provides:
- `add(Money $other): Money` — asserts same currency
- `subtract(Money $other): Money`
- `multiply(int|float $factor, int $roundingMode): Money`
- `convert(ExchangeRate $rate): Money`
- `format(): string` — locale-aware display string
- `toMinorUnits(): int`
- Static `fromMinorUnits(int $amount, string $currency): Money`

Never use `float` or `bcmath` directly in business logic — always go through this class.

---

## Entities & Key Fields

### `currencies`

| Column | Type | Notes |
|--------|------|-------|
| `code` | char(3) PK | ISO 4217 |
| `name` | string | e.g. "Omani Rial" |
| `symbol` | string | e.g. "OMR", "ر.ع." |
| `subunit` | smallint | Minor units per major unit (1000, 100, 1) |
| `decimal_places` | tinyint | Derived from subunit; stored for convenience |
| `active` | boolean | |

### `exchange_rates`

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `tenant_id` | FK | Rates are per-tenant (each tenant may have different base currency) |
| `from_currency_code` | char(3) | |
| `to_currency_code` | char(3) | |
| `rate` | decimal(20,10) | Exchange rate: 1 `from` = `rate` `to`. Stored as decimal, not as minor units |
| `effective_from` | date | Rate is valid from this date (inclusive) |
| `effective_to` | date | nullable; null = still current |
| `source` | string | 'manual' \| 'api' |
| `created_by` | FK → users | |
| `created_at` | timestamp | |

> Only one rate may be active per `(tenant_id, from_currency, to_currency)` pair at any given date. Enforced via a unique partial index or service-layer check.

### Rate locking on transactions

Every transaction table that records a foreign-currency amount must also store the rate used:

```sql
amount_foreign        bigint          -- minor units in foreign currency
currency_code_foreign char(3)
amount_base           bigint          -- minor units in tenant base currency
currency_code_base    char(3)
exchange_rate_id      FK → exchange_rates   -- rate locked at time of transaction
```

This ensures historical reports always reproduce the original base-currency value without recalculating.

---

## Business Rules

1. **No floats in storage.** Exchange rates are stored as `decimal(20,10)` in the `exchange_rates` table. Conversion math uses integer arithmetic where possible; when a ratio is needed, use `bcmath` inside the `Money` class with precision 10.
2. **Rate lookup.** When converting money, the system looks up the rate where `effective_from <= transaction_date` and `effective_to IS NULL OR effective_to >= transaction_date`. If no rate exists, the transaction is blocked and an error is shown.
3. **Symmetric rates.** Only one direction needs to be stored (e.g. OMR → USD). The inverse (USD → OMR) is computed as `1 / rate`. Both directions may be stored explicitly if the market spread matters; for MVP, one direction is enough.
4. **Rounding.** Always round half-up (PHP_ROUND_HALF_UP) for customer-facing amounts. Store the rounded value; do not accumulate fractional remainders across line items.
5. **Base currency for reports.** Each tenant has one base currency. All reports that aggregate across currencies convert to base currency using the rate on the report date (or transaction date for historical reports).
6. **Exchange gain/loss.** When a receivable or payable in a foreign currency is settled at a different rate than when it was recorded, the difference is booked as an exchange gain or loss to the Forex G/L account. This applies to Module 31 (Customer Ledger) and Module 33 (Supplier Ledger).
7. **POS multi-currency.** A customer may pay in a different currency than the branch operating currency (e.g. USD cash in an Oman branch). The POS uses the current exchange rate to convert and records both amounts.

---

## Workflows

### Adding or updating an exchange rate (manual)

1. Tenant admin navigates to Currency > Exchange Rates.
2. Selects from/to currency pair.
3. Enters rate and effective date.
4. System closes the previous rate's `effective_to` to the day before the new `effective_from`.
5. New rate record is created.
6. Audit log entry written.

### Automatic rate update (future / out of MVP scope)

A scheduled job fetches rates from an external provider (e.g. Open Exchange Rates, CurrencyLayer) and creates new rate records. Manual rates can override API rates for specific pairs.

### Converting an amount

```php
$rate = ExchangeRateRepository::rateOn($tenant, 'USD', 'OMR', $date);
$omr  = $usd->convert($rate);  // returns Money in OMR
```

---

## Permissions

| Permission | Who can |
|------------|---------|
| `exchange_rates.view` | All authenticated users |
| `exchange_rates.create` | Tenant admin, Accountant |
| `exchange_rates.update` | Tenant admin, Accountant |
| `currencies.manage` | Super-admin (adding new ISO currencies) |

---

## Dependencies

- Module 01 (Tenant & Branch): `tenant_id` on rates; base currency from tenant record.
- Module 02 (Country Packs): `currency_subunit` seeded from the pack.
- Module 08 (Audit Log): rate changes are audited.
- All financial modules (21, 31–36) consume this module for conversion and rate locking.

---

## Edge Cases

- **Same currency conversion.** `Money::convert()` where from == to returns the original money unchanged without looking up a rate.
- **Rate not found for date.** Hard block on saving the transaction; display which pair is missing so the user knows what to add.
- **Zero rate.** Rejected at validation (a zero rate would mean infinite conversion in one direction).
- **Historic rate query.** Reporting modules query rates as of the original transaction date, not today. Ensure `exchange_rates` is append-only (old rates are never deleted, only closed with `effective_to`).
- **PKR display.** Amounts are stored in paisa (subunit = 100) and displayed as rupees. `₨ 150.00` displays from 15000 stored paisa. Ensure all formatters divide by 100 and show 2 decimal places for PKR.

---

## Out of Scope (MVP)

- Automatic exchange rate feed / API integration.
- Spread (buy vs sell rate) — use a single mid-market rate.
- Cryptocurrency.
- Currency revaluation of open payables/receivables (period-end FX revaluation journals) — deferred to after Supplier Ledger is built.
