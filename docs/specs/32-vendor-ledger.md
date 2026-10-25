# Module 32 — Vendor Ledger

**Status:** Draft  
**Phase:** Finance  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Tracks payables to darzi/karigar vendors: what is owed per job, what has been paid, and the outstanding balance per vendor. The structure mirrors the Customer Ledger but for payables.

---

## Entities & Key Fields

### `vendor_ledger_entries`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `vendor_id` | FK | |
| `entry_date` | date | |
| `entry_type` | enum | `payable` \| `payment` \| `deduction` \| `adjustment` |
| `reference_type` | string | Polymorphic: VendorJob, etc. |
| `reference_id` | ULID | |
| `description` | string | |
| `debit_amount` | bigint | Payment made to vendor; minor units |
| `credit_amount` | bigint | Amount the shop owes (payable created); minor units |
| `currency_code` | char(3) | Vendor's currency |
| `exchange_rate_id` | FK | nullable |
| `base_debit_amount` | bigint | In tenant base currency |
| `base_credit_amount` | bigint | |
| `running_balance` | bigint | Positive = shop owes vendor; in base currency |
| `created_by` | FK → users | |
| `created_at` | timestamp | |

---

## Entry Types

| Event | Entry type | Debit (payment out) | Credit (payable created) |
|-------|-----------|---------------------|------------------------|
| Job passes QC | `payable` | — | `vendor_rate_amount` |
| Payment made to vendor | `payment` | `payment_amount` | — |
| Deduction (quality issue, agreed) | `deduction` | — (reduces payable) | negative credit |
| Manual correction | `adjustment` | or credit | as needed |

---

## Business Rules

1. **Payable created on QC pass.** When a `vendor_job` moves to `qc_passed`, a `vendor_ledger_entries` record of type `payable` is created automatically.
2. **Payment to vendor.** When the shop pays a vendor (cash, bank transfer), a `payment` entry is created and cash is recorded in Module 35.
3. **Vendor currency.** If the vendor is paid in a different currency than the tenant base, `exchange_rate_id` is set and `base_debit_amount` records the base currency equivalent.
4. **Vendor statement.** A statement showing all payables and payments within a date range, with outstanding balance, is printable as a vendor account statement.
5. **Deduction.** If a deduction is agreed (e.g. vendor agreed to a discount on a QC-failed piece), a `deduction` entry reduces the outstanding balance. Requires admin approval.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `vendor_ledger.view` | Manager, Admin, Accountant |
| `vendor_ledger.pay` | Admin, Accountant |
| `vendor_ledger.deduction` | Admin |

---

## Dependencies

- Module 13 (Vendors): vendor master.
- Module 25 (Vendor Job Management): payables created on QC pass.
- Module 04 (Currency): foreign currency payables.
- Module 35 (Cash Management): payment outflows.

---

## Out of Scope (MVP)

- Automated payment runs (batch payment to multiple vendors).
- Vendor-side statement portal.
