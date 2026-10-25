# Module 28 — Delivery & Pickup

**Status:** Draft  
**Phase:** Sales & Services  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages the in-shop collection and home delivery of stitching orders and ironing orders. Tracks uncollected orders, aging, and reminder triggers. Home delivery (courier integration for ecommerce) is in Module 40.

---

## Entities & Key Fields

### `delivery_assignments`

For home delivery (driver-based, local delivery — not courier).

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `order_type` | enum | `stitching` \| `ironing` \| `alteration` |
| `order_id` | ULID | Polymorphic reference |
| `customer_id` | FK | |
| `delivery_address` | JSON | {street, area, city} |
| `scheduled_date` | date | |
| `assigned_to` | FK → employees | Driver/delivery employee |
| `status` | enum | `scheduled` \| `out_for_delivery` \| `delivered` \| `failed` |
| `delivered_at` | timestamp | nullable |
| `failure_reason` | text | nullable |
| `collected_amount` | bigint | If cash on delivery |

### `uncollected_order_log`

Tracks when reminders were sent for uncollected orders.

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `order_type` | enum | |
| `order_id` | ULID | |
| `reminder_number` | tinyint | 1st, 2nd, 3rd reminder |
| `sent_at` | timestamp | |
| `channel` | string | whatsapp \| sms |

---

## Business Rules

1. **Uncollected detection.** A scheduled job runs nightly. It finds all stitching and ironing orders in `ready` or `done` status where `promised_date + uncollected_days ≤ today`. These are transitioned to `uncollected` status and a first reminder is queued (Module 37).
2. **Repeat reminders.** Every `uncollected_repeat_days` thereafter, another reminder is sent (up to 3 times by default; configurable). After the 3rd reminder, the order appears on the "aging" report for staff to follow up by phone.
3. **Aging report.** Shows all uncollected orders grouped by days overdue: 0–7 days, 8–30 days, 31+ days. Includes outstanding balance due.
4. **In-shop collection.** When a customer collects in the shop, the cashier opens the order, confirms pieces, collects balance, and marks as `delivered`. This is the same flow as covered in Module 23 and 27; this module adds the formal delivery tracking layer.
5. **Home delivery.** Optional. If the shop offers home delivery (own driver, not courier), a `delivery_assignment` is created. The driver is assigned, marks `out_for_delivery` on departure, and `delivered` on completion. Cash collected is logged.

---

## Workflow: Uncollected reminder

```
Nightly job runs
    └─ Finds orders: status = ready/done, promised_date + N days ≤ today
         └─ Transitions to uncollected
              └─ Queues 1st WhatsApp/SMS reminder
                   └─ Every uncollected_repeat_days: sends next reminder (up to 3)
                        └─ After 3rd reminder: appears on aging report
```

---

## Permissions

| Permission | Who can |
|------------|---------|
| `delivery.view` | Manager, Admin |
| `delivery.assign` | Manager, Admin |
| `delivery.update_status` | Driver (own assignments), Manager, Admin |
| `uncollected.view` | Manager, Admin |

---

## Dependencies

- Module 23 (Stitching Orders): source of orders.
- Module 27 (Ironing): source of ironing orders.
- Module 09 (Shop Settings): `uncollected_days`, `uncollected_repeat_days`.
- Module 14 (Employees): driver assignment.
- Module 37 (Notifications): reminder sending.

---

## Out of Scope (MVP)

- Courier integration for home delivery (Module 40).
- Delivery route optimisation.
- Customer-facing delivery tracking link.
- Late fees for uncollected orders (data is available; policy not implemented in MVP).
