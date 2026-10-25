# Module 14 — Employees

**Status:** Draft  
**Phase:** Parties  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages the shop's own staff (not darzi/karigar, who are external vendors in Module 13). Covers HR basics needed for day-to-day shop operations: employee profiles, attendance, salary structure, advances, and commission. WPS (Wage Protection System) export for Oman is deferred.

Scope: employee records, attendance, salary, advances, and commission on sales. Full payroll processing is out of MVP scope — this module provides the data; payroll calculations are a later phase.

---

## Entities & Key Fields

### `employees`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `branch_id` | FK | Primary branch; can be assigned shifts at other branches |
| `user_id` | FK → users | nullable; if the employee also has a system login |
| `name` | string(255) | |
| `code` | string(20) | Employee number; unique per tenant |
| `phone` | string | E.164 |
| `email` | string | nullable |
| `id_type` | enum | `national_id` \| `passport` \| `resident_card` |
| `id_number` | string | Encrypted at rest |
| `nationality` | char(2) | ISO 3166-1 alpha-2 |
| `date_of_birth` | date | nullable |
| `join_date` | date | |
| `end_date` | date | nullable; last day of employment |
| `designation` | string | e.g. "Cashier", "Counter Staff", "Supervisor" |
| `department` | string | nullable; e.g. "Sales", "Stitching", "Accounts" |
| `salary_amount` | bigint | Monthly base salary in minor units |
| `salary_currency_code` | char(3) | |
| `commission_type` | enum | `none` \| `percent_of_sales` \| `fixed_per_piece` |
| `commission_rate` | decimal(8,4) | Percentage or fixed amount; depends on `commission_type` |
| `active` | boolean | |
| `notes` | text | nullable |
| `created_at` / `updated_at` | timestamps | |

### `employee_attendance`

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `employee_id` | FK | |
| `branch_id` | FK | |
| `date` | date | |
| `check_in` | time | nullable |
| `check_out` | time | nullable |
| `status` | enum | `present` \| `absent` \| `leave` \| `half_day` \| `holiday` |
| `notes` | text | nullable |
| `recorded_by` | FK → users | |

### `employee_advances`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `employee_id` | FK | |
| `amount` | bigint | Minor units |
| `currency_code` | char(3) | |
| `given_date` | date | |
| `reason` | text | nullable |
| `recovered_amount` | bigint | How much has been deducted from salary so far |
| `fully_recovered` | boolean | |
| `approved_by` | FK → users | |
| `created_at` | timestamp | |

---

## Business Rules

1. **Employee ↔ User link.** An employee may or may not have a system login. If they do, `employee_id` on the `users` table links the two records. Deactivating an employee does not automatically deactivate their user account — this must be done separately (system will warn).
2. **Attendance.** Attendance is recorded daily per branch. If an employee works across branches in a day, two records are created (one per branch).
3. **Commission calculation.** Commission is calculated at salary-run time (outside MVP) based on: `percent_of_sales` = sum of attributed sales × rate; `fixed_per_piece` = count of completed stitching pieces × rate. Sales are attributed to the cashier who processed them (`created_by` on the sale).
4. **Advance recovery.** When processing salary (future module), outstanding advances are deducted from the net salary. `recovered_amount` is incremented; `fully_recovered` is set to true when `recovered_amount = amount`.
5. **End date.** When `end_date` is set, the employee is considered separated. `active` is set to false on that date. Historical records (attendance, advances, commission) are retained.
6. **ID number encryption.** Encrypted at rest. Audit log records access.

---

## Workflows

### Onboarding an employee

1. Admin creates employee record: personal details, join date, designation, salary, commission structure.
2. Optionally creates a linked user account (Module 07) with the appropriate role.
3. Employee appears in attendance sheets from their `join_date`.

### Recording attendance

1. Manager (or admin) opens the attendance screen for the branch and date.
2. Each employee is listed; status is marked (present, absent, leave, etc.) with optional check-in/check-out times.
3. Records saved; attendance report updated.

### Issuing an advance

1. Employee requests advance.
2. Manager/Admin records it: amount, date, reason.
3. Approval recorded (approver's user ID).
4. Cash expense entry created in Module 34 (Expenses) to record the cash outflow.

---

## Commission Attribution

When a cashier processes a POS sale or a stitching order is marked complete:
- The `created_by` (or `processed_by`) user is linked to their employee record.
- The sale amount or piece count is accumulated in a commissions summary table (monthly).
- At salary run time (future payroll module), accumulated commission is added to base salary.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `employees.view` | Manager, Admin |
| `employees.create` | Admin |
| `employees.update` | Admin |
| `employees.deactivate` | Admin |
| `attendance.view` | Manager, Admin |
| `attendance.manage` | Manager, Admin |
| `advances.view` | Manager, Admin, Accountant |
| `advances.create` | Manager, Admin |
| `advances.approve` | Admin |

---

## Dependencies

- Module 01 (Tenant & Branch): employee is scoped to a branch.
- Module 04 (Currency): salary and advances in minor units.
- Module 07 (Users): employee may be linked to a user account.
- Module 08 (Audit Log): ID number access, advance approvals audited.
- Module 34 (Expenses): advance payment creates an expense entry.
- Module 35 (Cash Management): advance payout recorded in cash book.

---

## Edge Cases

- **Employee with no system login.** Attendance and advances still work; commission is still tracked via their linked user if they have one later. If no user link exists, commission cannot be auto-attributed — manual entry required.
- **Employee working in multiple branches in one day.** Two attendance records for the same date, different `branch_id`. Reports show full-day across both.
- **Salary in a different currency.** Foreign workers (e.g. Omani shop employing expats in PKR). Salary stored in the agreed currency; converted to base currency in reports at the payslip date's exchange rate.
- **Advance larger than one month's salary.** The system allows this; it is an admin/business decision, not a system constraint. The warning is shown (advance > 1× salary) but not blocking.

---

## Out of Scope (MVP)

- Full payroll processing (salary slip generation, deductions, end-of-service).
- WPS (Wage Protection System) export for Oman — the data structure supports it; the export is deferred.
- Leave management (annual leave, sick leave entitlement tracking).
- Biometric / fingerprint attendance integration.
- Employee self-service (view payslip, apply for leave).
- Employee contracts and document storage.
