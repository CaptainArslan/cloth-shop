# Module 30 — Capacity & Booking Control

**Status:** Draft  
**Phase:** Planning  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Prevents overbooking of stitching capacity by enforcing daily order limits and seasonal cutoffs (Eid, peak season). Integrates with the calendar (Module 29) to warn or block new bookings when capacity is reached.

---

## Business Rules

1. **Daily order limit.** `calendar.max_orders_per_day` (from Shop Settings) limits how many new stitching order lines can promise a given delivery date. When the limit is reached, the system warns (soft block by default; hard block configurable). The cashier can override with manager PIN.
2. **Eid cutoff.** When the Eid date is confirmed in Module 02, the system automatically blocks new stitching orders promising delivery after `eid_date - eid_cutoff_days_before`. Staff are warned from one week before the cutoff. The cutoff is branch-configurable.
3. **Weekend and holiday blocking.** `CalendarService::isWorkingDay()` prevents promised dates from falling on weekends or holidays. If a date is selected that is a holiday, the system suggests the next working day.
4. **Vendor capacity.** When assigning a vendor job (Module 25), the system checks the vendor's `max_pieces_per_day` against existing assignments for the expected return date. Over-capacity triggers a warning (not a hard block in MVP).
5. **Peak season flag.** A manual "peak season" flag can be set by admin. When active, the `max_orders_per_day` is reduced to a configurable percentage of the normal limit (e.g. 60% of normal when peak = true). This handles pre-Eid rush management.

---

## Entities & Key Fields

### `capacity_overrides`

For date-specific manual adjustments to the daily limit.

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `branch_id` | FK | |
| `date` | date | |
| `max_orders` | smallint | Override for this specific date (0 = fully blocked) |
| `reason` | string | nullable |
| `created_by` | FK → users | |

---

## Workflows

### Blocking a specific date (e.g. shop is closed for a private event)

Admin opens calendar → selects date → sets `capacity_override.max_orders = 0` with reason "Private event". Any attempt to promise delivery on that date is hard-blocked with the reason shown.

### Eid cutoff activation

1. Eid date is confirmed in Module 02 and stored.
2. System automatically computes `eid_date - eid_cutoff_days_before` as the cutoff date.
3. Any new order promising delivery after the cutoff is blocked (hard block for Eid — not a soft warning).
4. A banner is shown at the POS: "Eid orders are closed. Accepting post-Eid orders only."

---

## Permissions

| Permission | Who can |
|------------|---------|
| `capacity.view` | Manager, Admin |
| `capacity.manage_overrides` | Manager, Admin |
| `capacity.override_limit` | Manager (PIN), Admin |

---

## Dependencies

- Module 02 (Country Packs): holidays, Eid date, weekend days.
- Module 09 (Shop Settings): `max_orders_per_day`, `eid_cutoff_days_before`.
- Module 23 (Stitching Orders): checks capacity before saving a promised date.
- Module 25 (Vendor Jobs): checks vendor capacity before issuing a job.
- Module 29 (Calendar): displays capacity fill per day.

---

## Out of Scope (MVP)

- Per-garment-type capacity (e.g. max 20 dishdashas and 10 abayas per day independently).
- Waiting list management.
- Automated reallocation when a vendor's capacity is exceeded.
