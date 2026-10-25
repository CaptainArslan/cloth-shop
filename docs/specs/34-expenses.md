# Module 34 — Expenses

**Status:** Draft  
**Phase:** Finance  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Records all operational expenditures that are not purchase invoices (supplier payments) or vendor payments: rent, utilities, petty cash expenses, marketing costs, employee advances paid out, and any other outgoings. Feeds into the P&L module (Module 36).

---

## Entities & Key Fields

### `expense_categories`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `name` | JSON | Translatable |
| `code` | string | e.g. `rent`, `utilities`, `marketing`, `petty_cash` |
| `affects_pl` | boolean | True for all normal expenses; false for internal transfers |
| `parent_id` | FK → self | nullable; for sub-categories |

### `expenses`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `expense_number` | string | Auto-generated |
| `expense_category_id` | FK | |
| `description` | string | |
| `amount` | bigint | Minor units |
| `currency_code` | char(3) | |
| `expense_date` | date | |
| `payment_method_id` | FK | How it was paid |
| `account_id` | FK | Which cash account / bank the payment came from (Module 35) |
| `receipt_reference` | string | nullable; supplier receipt or voucher number |
| `tax_amount` | bigint | nullable; if tax is recoverable (input tax) |
| `approved_by` | FK → users | nullable |
| `created_by` | FK → users | |
| `created_at` | timestamp | |

---

## Business Rules

1. **Every expense hits an account.** All expenses must be associated with an `account_id` from Module 35 (the cash register, petty cash box, or bank account the money came from). This keeps the cash book accurate.
2. **Approval threshold.** Expenses above a configurable amount (`expenses.approval_threshold`) require manager or admin approval before they are included in the P&L. Below the threshold, cashier/staff can record them directly.
3. **Employee advance as expense.** When an advance is paid to an employee (Module 14), the cash outflow is recorded as an expense of category `employee_advance`. When recovered from salary, a reverse entry is made.
4. **Recurring expenses.** Some expenses (rent, subscriptions) recur monthly. MVP: entered manually each period. A recurring template (auto-create) is a later feature.
5. **Input tax on expenses.** If the expense carries a tax component that is recoverable (e.g. VAT on business expenses in Oman), `tax_amount` is recorded separately. This feeds the VAT return (fiscal module — future).

---

## Workflow

1. Staff records expense: category, description, amount, date, payment method, account.
2. If above approval threshold: pending state; manager/admin reviews and approves.
3. Approved expense: cash book entry created in Module 35; expense appears in P&L.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `expenses.create` | Cashier (below threshold), Manager, Admin |
| `expenses.approve` | Manager, Admin |
| `expenses.view` | Manager, Admin, Accountant |
| `expense_categories.manage` | Admin |

---

## Dependencies

- Module 35 (Cash & Accounts): payment debits the relevant account.
- Module 36 (P&L): expenses feed the cost side.
- Module 14 (Employees): advance expenses referenced.

---

## Out of Scope (MVP)

- Expense reports (employee submits, manager approves, reimburses) — not applicable to this shop model.
- Recurring expense auto-creation.
- Receipt image upload.
- Credit card expense reconciliation.
