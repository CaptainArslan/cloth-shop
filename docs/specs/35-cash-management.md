# Module 35 — Accounts & Cash Management

**Status:** Draft  
**Phase:** Finance  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages the flow of money through the shop's cash registers, petty cash boxes, and bank accounts. Tracks every cash in and cash out, supports day-end closing (reconciliation of physical cash against system totals), and provides the day book. Also handles cash transfers between accounts.

---

## Entities & Key Fields

### `accounts`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `name` | string | e.g. "Main Cash Register", "Petty Cash", "HSBC Oman Current" |
| `type` | enum | `cash_register` \| `petty_cash` \| `bank` \| `mobile_wallet` |
| `currency_code` | char(3) | |
| `opening_balance` | bigint | Minor units; set when account is created |
| `current_balance` | bigint | Running total; updated on each transaction |
| `active` | boolean | |
| `is_default_pos` | boolean | Which account POS cash payments go to |

### `account_transactions`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `account_id` | FK | |
| `transaction_date` | date | |
| `transaction_type` | enum | `sale_payment` \| `expense` \| `vendor_payment` \| `supplier_payment` \| `customer_refund` \| `transfer_in` \| `transfer_out` \| `day_open` \| `day_close` \| `adjustment` |
| `reference_type` | string | Polymorphic |
| `reference_id` | ULID | nullable |
| `description` | string | |
| `debit_amount` | bigint | Money in (increases balance) |
| `credit_amount` | bigint | Money out (decreases balance) |
| `running_balance` | bigint | |
| `created_by` | FK → users | |
| `created_at` | timestamp | |

### `cash_sessions`

One record per business day per cash register per branch.

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `branch_id` | FK | |
| `account_id` | FK | |
| `opened_by` | FK → users | |
| `closed_by` | FK → users | nullable |
| `opened_at` | timestamp | |
| `closed_at` | timestamp | nullable |
| `opening_float` | bigint | Physical cash counted at open |
| `expected_closing` | bigint | Computed from transactions |
| `actual_closing` | bigint | Physical cash counted at close |
| `variance` | bigint | `actual - expected`; negative = shortage |
| `variance_reason` | text | nullable |
| `status` | enum | `open` \| `closed` |

### `account_transfers`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `from_account_id` | FK | |
| `to_account_id` | FK | |
| `amount` | bigint | |
| `currency_code` | char(3) | |
| `transfer_date` | date | |
| `reference` | string | nullable |
| `created_by` | FK → users | |

---

## Business Rules

1. **Every financial movement hits an account.** POS payments, expense payments, vendor payments, supplier payments, customer refunds — all create `account_transactions` in the same DB transaction as the source record.
2. **Running balance maintained.** `account_transactions.running_balance` and `accounts.current_balance` are updated at insert time within the same transaction. This gives instant account balance without summing the log.
3. **Day-end closing.** Before closing, the cashier counts the physical cash in the register and enters the actual amount. The system computes `expected_closing = opening_float + cash_in - cash_out` for the day. The variance is recorded and a closing report is printed.
4. **Variance handling.** A variance within a configurable tolerance is auto-approved. Above tolerance: manager must review and enter a reason before the session can be closed.
5. **Cash session required.** Cash payments cannot be recorded unless an active cash session is open for that register. Opening the POS automatically opens a session if none is active.
6. **Account transfer.** When cash is moved from the register to petty cash or to the bank, an `account_transfer` is created: a `transfer_out` transaction on the source account and a `transfer_in` on the destination.
7. **Non-cash accounts (bank, card).** Card payments are recorded against a card account. The physical bank reconciliation (matching card settlements to bank statements) is manual in MVP — there is no automated bank feed.

---

## Day-End Closing Workflow

1. Cashier opens "Close Day".
2. System shows: expected cash, expected card totals by payment method, list of all transactions for the day.
3. Cashier counts physical cash and enters the amount.
4. System shows variance.
5. If variance within tolerance: close automatically. If above: manager enters reason.
6. Closing summary report printed. Cash session marked `closed`.
7. Next day: cashier opens a new session and enters the opening float (usually the closing balance left in the register).

---

## Permissions

| Permission | Who can |
|------------|---------|
| `cash_sessions.open` | Cashier, Manager, Admin |
| `cash_sessions.close` | Cashier, Manager, Admin |
| `cash_sessions.close_with_variance` | Manager, Admin |
| `accounts.view` | Manager, Admin, Accountant |
| `accounts.manage` | Admin |
| `account_transfers.create` | Manager, Admin, Accountant |
| `expenses.pay` | Cashier (petty cash, below threshold), Manager, Admin |

---

## Dependencies

- Module 04 (Currency): amounts in minor units.
- Module 08 (Audit Log): all account transactions and closing events.
- Module 21 (POS): sale payments feed account transactions.
- Module 32, 33 (Vendor/Supplier Ledger): outgoing payments.
- Module 34 (Expenses): expense payments.
- Module 36 (P&L): account balances feed the balance sheet.

---

## Out of Scope (MVP)

- Automated bank feed / bank statement import.
- Bank reconciliation workflow.
- Multi-currency accounts (each account has one currency; FX conversion on cross-currency transfers is manual).
- Automated cash session opening at a set time.
