# Module 19 — Stock Audit

**Status:** Draft  
**Phase:** Inventory  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Periodic physical counting of rolls and pieces to reconcile the system's book stock against what is actually on the shelves. Discrepancies create adjustment records that correct `remaining_length` (for rolls) or piece counts, with mandatory reasons and audit trail.

---

## Entities & Key Fields

### `stock_audits`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `audit_number` | string | Auto-generated |
| `status` | enum | `in_progress` \| `completed` \| `cancelled` |
| `started_at` | timestamp | |
| `completed_at` | timestamp | nullable |
| `conducted_by` | FK → users | |
| `approved_by` | FK → users | nullable |
| `notes` | text | nullable |

### `stock_audit_lines`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `stock_audit_id` | FK | |
| `fabric_roll_id` | FK | |
| `book_length` | decimal(8,3) | System's `remaining_length` at time of audit |
| `counted_length` | decimal(8,3) | Physically measured |
| `variance` | decimal(8,3) | `counted - book`; negative = shortage |
| `variance_reason` | text | nullable; required when |variance| > threshold |
| `adjustment_applied` | boolean | Whether the `roll_cuts` adjustment was written |

---

## Business Rules

1. **Freeze-free audit (MVP).** The system does not lock stock during the audit. Cuts made during the audit are visible and the `book_length` is snapshotted at the moment each line is added to the audit sheet.
2. **Variance threshold.** A configurable setting (`audit.variance_reason_threshold`) defines the variance length above which a reason is mandatory before the audit can be completed. Default: 0.10m.
3. **Adjustment record.** On audit completion and approval, for each line with a non-zero variance, a `roll_cuts` record with `cut_type = stock_audit` is created. If variance is negative (shortage), the cut length is positive (reducing stock). If positive (surplus), a negative cut is recorded (increasing stock).
4. **Approval required.** Audit adjustments require a second user (manager or admin) to approve before they are written. The auditing staff member cannot self-approve.
5. **Historical snapshots.** Previous audit records are never deleted. They form a history of periodic stock verification.

---

## Workflow

1. Manager initiates audit for a branch (optionally filtered to a product or category).
2. System generates an audit sheet listing all `in_stock` and `remnant` rolls with their book lengths.
3. Staff physically measure each roll and enter counted lengths.
4. System highlights variances above the threshold and prompts for reasons.
5. Manager reviews and approves.
6. System writes adjustment `roll_cuts` records and updates `remaining_length`.
7. Audit report saved.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `stock_audits.create` | Manager, Admin |
| `stock_audits.conduct` | Manager, Admin (and designated staff) |
| `stock_audits.approve` | Admin (cannot be the same user who conducted) |

---

## Dependencies

- Module 16 (Roll Inventory): adjustments written as `roll_cuts`.
- Module 08 (Audit Log): audit events logged.

---

## Out of Scope (MVP)

- Blind counting (staff count without seeing book values — prevents bias).
- Cycle counting (continuous rolling audit of subsets).
- Automated discrepancy alerts (stock below a threshold).
