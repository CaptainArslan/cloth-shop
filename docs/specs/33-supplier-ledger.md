# Module 33 — Supplier Ledger

**Status:** Draft  
**Phase:** Finance  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Tracks payables to fabric suppliers and other inventory suppliers. Handles foreign-currency invoices, period-end currency revaluation, and payment tracking. Mirrors the Vendor Ledger in structure but for goods purchases rather than services.

---

## Entities & Key Fields

### `supplier_ledger_entries`

Same structure as `vendor_ledger_entries` but with `supplier_id` instead of `vendor_id`:

| Column | Type | Notes |
|--------|------|-------|
| `entry_type` | enum | `invoice` \| `payment` \| `credit_note` \| `adjustment` |
| `currency_code` | char(3) | Supplier invoice currency |
| `exchange_rate_id` | FK | Rate locked at time of GRN confirmation |
| `base_credit_amount` | bigint | Foreign invoice amount in base currency at locked rate |
| … | | (same columns as vendor_ledger_entries) |

---

## Entry Types

| Event | Entry type | Effect |
|-------|-----------|--------|
| GRN confirmed | `invoice` | Creates payable; base currency amount locked at GRN date rate |
| Payment made to supplier | `payment` | Reduces payable |
| Purchase return accepted | `credit_note` | Reduces payable |
| Manual correction | `adjustment` | |

---

## Business Rules

1. **Foreign currency payables.** If a supplier invoices in USD (e.g. a Dubai fabric supplier), the payable is recorded in USD. The base currency equivalent is locked at the GRN date rate. When payment is made at a different rate, the difference is recorded as exchange gain/loss in Module 35.
2. **Payment due date.** `due_date = invoice_date + supplier.payment_terms_days`. The aging report (Module 38) groups outstanding payables by overdue status using this date.
3. **Partial payments.** Multiple payment entries can be applied against one invoice. The running balance tracks what is still outstanding.
4. **Currency revaluation (future).** At period end, open foreign-currency payables should be revalued at the current rate and the difference booked as a foreign exchange gain/loss. This is deferred to a later phase; the data structure supports it.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `supplier_ledger.view` | Manager, Admin, Accountant |
| `supplier_ledger.pay` | Admin, Accountant |
| `supplier_ledger.adjustment` | Admin |

---

## Dependencies

- Module 12 (Suppliers): supplier master.
- Module 17 (Purchases): GRN confirmation creates invoice entries.
- Module 04 (Currency): foreign currency amounts and rate locking.
- Module 35 (Cash & Accounts): payment outflows; exchange gain/loss entries.

---

## Out of Scope (MVP)

- Period-end FX revaluation.
- Early payment discounts.
- Supplier statement reconciliation.
