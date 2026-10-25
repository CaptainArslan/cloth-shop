# Module 31 — Customer Ledger

**Status:** Draft  
**Phase:** Finance  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Tracks every financial transaction with each customer: sales, payments, credit notes, and advance receipts. The ledger is the authoritative record of what each customer owes. The `credit_balance` on the `customers` record is a denormalised cache; the ledger is the source of truth.

---

## Entities & Key Fields

### `customer_ledger_entries`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `customer_id` | FK | |
| `entry_date` | date | |
| `entry_type` | enum | `sale` \| `payment` \| `credit_note` \| `advance` \| `adjustment` |
| `reference_type` | string | Polymorphic: Sale, StitchingOrder, SaleReturn, etc. |
| `reference_id` | ULID | |
| `description` | string | Human-readable; e.g. "Invoice #ST-00042" |
| `debit_amount` | bigint | Amount the customer owes (increases balance); minor units |
| `credit_amount` | bigint | Amount applied against the balance; minor units |
| `currency_code` | char(3) | |
| `exchange_rate_id` | FK | nullable; if entry in non-base currency |
| `base_debit_amount` | bigint | Converted to tenant base currency |
| `base_credit_amount` | bigint | |
| `running_balance` | bigint | Running balance in base currency at this entry |
| `created_by` | FK → users | |
| `created_at` | timestamp | |

---

## Entry Types

| Event | Entry type | Debit | Credit |
|-------|-----------|-------|--------|
| Sale completed (credit portion) | `sale` | `balance_due` | — |
| Payment received | `payment` | — | `payment_amount` |
| Advance received at order | `advance` | — | `advance_amount` |
| Return/refund issued | `credit_note` | — | `refund_amount` |
| Balance write-off (admin) | `adjustment` | or credit | as needed |

---

## Business Rules

1. **Ledger is append-only.** No entries are ever updated or deleted. Corrections are made via new `adjustment` entries with a reference to the original and a reason.
2. **Running balance.** `running_balance` is computed at insert time: previous running balance ± this entry. This allows fast balance lookup without summing the whole ledger.
3. **Customer.credit_balance sync.** After every ledger insert, `customers.credit_balance` is updated to match the latest `running_balance`. This update is in the same DB transaction as the ledger insert.
4. **Multi-currency.** Entries can be in any currency; each carries `exchange_rate_id` and `base_*_amount`. The `running_balance` is always in the tenant base currency.
5. **Exchange gain/loss on settlement.** If a debit was recorded in OMR equivalent of USD 100 (at rate 0.385), and the customer later pays USD 100 at a different rate, the difference is booked as an exchange gain/loss adjustment entry.
6. **Statement generation.** A customer statement lists all ledger entries between two dates with a running balance, usable as a customer-facing account statement.

---

## Workflow: Credit sale → payment

1. Sale completed; `balance_due = OMR 15.000`.
2. Ledger entry: type=`sale`, debit=15000 (baisa), running_balance increases by 15000.
3. Customer comes in next day and pays OMR 10.000 cash.
4. Ledger entry: type=`payment`, credit=10000, running_balance decreases.
5. `customers.credit_balance` updated to 5000 (baisa).
6. Receipt printed showing remaining balance.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `customer_ledger.view` | Manager, Admin, Accountant |
| `customer_ledger.collect_payment` | Cashier, Manager, Admin |
| `customer_ledger.adjustment` | Admin, Accountant |
| `customer_statement.print` | Cashier, Manager, Admin |

---

## Dependencies

- Module 10 (Customers): `credit_balance` denormalised.
- Module 04 (Currency): multi-currency entries.
- Module 21 (POS), 23 (Stitching Orders): create sale entries.
- Module 22 (Returns): create credit note entries.

---

## Out of Scope (MVP)

- Automated dunning (reminder emails for overdue balances) — covered by Module 37 for uncollected orders; general debtor dunning is a later feature.
- Bad debt write-off workflow (the adjustment entry type handles it; formal workflow deferred).
- Aging analysis by invoice (the aging report in Module 38 shows totals; invoice-level aging is a later feature).
