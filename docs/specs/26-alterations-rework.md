# Module 26 — Alterations & Rework

**Status:** Draft  
**Phase:** Sales & Services  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Handles post-delivery alterations and rework for garments originally stitched by the shop. Free alterations within the defined window (from Module 23 settings), and charged alterations after that window. A separate alteration order is created for each request, linking back to the original stitching order line.

Note: vendor rework during QC (before delivery) is handled in Module 25. This module is specifically for customer-requested changes after delivery.

---

## Entities & Key Fields

### `alteration_orders`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `alteration_number` | string | |
| `customer_id` | FK | |
| `original_stitching_order_line_id` | FK | nullable; if alteration is on a previously stitched order |
| `is_free` | boolean | True if within the free alteration window |
| `status` | enum | `received` \| `issued_to_vendor` \| `done` \| `delivered` |
| `received_date` | date | |
| `promised_date` | date | |
| `delivery_date` | date | nullable |
| `description` | text | What needs to be altered |
| `charge_amount` | bigint | 0 if is_free; minor units |
| `currency_code` | char(3) | |
| `vendor_id` | FK | nullable; if outsourced |
| `vendor_rate_amount` | bigint | nullable |
| `created_by` | FK → users | |

---

## Business Rules

1. **Free window check.** When creating an alteration for a known order line, the system checks: `delivery_date + alteration_free_days ≥ today`. If yes, `is_free = true` and `charge_amount = 0`. If outside the window, the cashier enters a charge.
2. **No original order.** If a customer brings a garment not originally stitched by the shop, `original_stitching_order_line_id = null`. This is always a charged alteration.
3. **QR tag.** A new QR tag is printed for the alteration order and attached to the garment so it can be tracked through the shop.
4. **Vendor assignment.** If the alteration is sent to a vendor (same or different from original), a `vendor_job` is created (simplified — Module 25 flow applies).
5. **Charge collection.** If the alteration is charged, payment is collected at delivery (same as stitching order balance collection flow).

---

## Workflow

1. Customer brings garment in for alteration.
2. Staff opens new alteration order, selects customer, optionally links to original order line.
3. System auto-checks free window; sets `is_free` accordingly.
4. Staff describes what needs to be done.
5. If sent to vendor, job slip issued. If done in-house, status tracked manually.
6. On completion, status → `done`.
7. Customer notified (Module 37). Customer collects; payment collected if applicable; status → `delivered`.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `alterations.create` | Cashier, Manager, Admin |
| `alterations.charge` | Manager, Admin (to override free window) |
| `alterations.deliver` | Cashier, Manager, Admin |

---

## Dependencies

- Module 23 (Stitching Orders): links to original order line for free window calculation.
- Module 25 (Vendor Job Management): if alteration is outsourced.
- Module 37 (Notifications): ready notification to customer.

---

## Out of Scope (MVP)

- Alteration of garments not stitched by the shop but from a known external source.
- Alterations on ready-made garments purchased at the shop.
- Alteration rate card (MVP is manual charge entry).
