# Module 23 — Stitching Orders

**Status:** Draft  
**Phase:** Sales & Services  
**Last updated:** 2026-10-25

---

## Purpose & Scope

The core operational module for the stitching business. Covers the full lifecycle of a stitching order: booking, measurement selection, fabric assignment, advance collection, tag printing, vendor assignment, progress tracking, delivery, and uncollected-order management. Each order is multi-line (multiple garments for different family members).

Scope: order creation through delivery. Vendor job management (issue slips, QC) is in Module 25. Alterations and rework are in Module 26.

---

## Entities & Key Fields

### `stitching_orders`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `order_number` | string | From invoice sequence (`stitching_prefix`) |
| `customer_id` | FK | |
| `booking_date` | date | |
| `promised_delivery_date` | date | |
| `status` | enum | `booked` \| `in_progress` \| `partial_ready` \| `ready` \| `delivered` \| `partially_delivered` \| `uncollected` \| `cancelled` |
| `subtotal_amount` | bigint | Sum of line amounts before discount and tax |
| `discount_amount` | bigint | |
| `tax_amount` | bigint | |
| `total_amount` | bigint | |
| `advance_amount` | bigint | Collected at booking |
| `advance_payment_method_id` | FK | |
| `balance_due` | bigint | `total - advance - subsequent_payments` |
| `currency_code` | char(3) | |
| `is_urgent` | boolean | |
| `urgency_surcharge_amount` | bigint | Applied if is_urgent = true |
| `fiscal_status` / `fiscal_number` | string | From Module 06 |
| `notes` | text | nullable |
| `created_by` / `delivered_by` | FK → users | |
| `created_at` / `updated_at` | timestamps | |

### `stitching_order_lines`

One record per garment (e.g. one dishdasha for the father, one for the son = two lines).

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `stitching_order_id` | FK | |
| `line_number` | tinyint | Display order |
| `customer_family_member_id` | FK | Who this garment is for |
| `garment_template_id` | FK | |
| `measurement_set_id` | FK → customer_measurements | The measurement version selected at booking |
| `measurement_snapshot` | JSON | Copy of the measurement values at booking |
| `style_notes` | text | nullable; collar style, pocket details, embroidery, etc. |
| `fabric_source` | enum | `shop` \| `customer_own` |
| `fabric_roll_id` | FK | nullable; required if fabric_source = shop |
| `fabric_length` | decimal(8,3) | Metres allocated to this line |
| `unit_price_amount` | bigint | Stitching charge for this garment |
| `discount_percent` | decimal(5,2) | |
| `total_amount` | bigint | |
| `status` | enum | `pending` \| `issued_to_vendor` \| `in_stitching` \| `stitching_done` \| `qc_passed` \| `ready` \| `delivered` \| `rework` |
| `vendor_id` | FK | nullable; assigned vendor (Module 25) |
| `promised_date` | date | Per-line promise if different from order level |
| `qr_tag_printed` | boolean | |
| `delivery_date` | date | nullable; actual date delivered to customer |

### `stitching_order_payments`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `stitching_order_id` | FK | |
| `amount` | bigint | |
| `currency_code` | char(3) | |
| `payment_method_id` | FK | |
| `payment_date` | date | |
| `is_advance` | boolean | True for the booking advance |
| `collected_by` | FK → users | |

---

## Order Status Lifecycle

```
booked
  └─ (vendor issued) → in_progress
       └─ (some lines ready) → partial_ready
            └─ (all lines ready) → ready
                 └─ (customer collects) → delivered
                      OR (not collected after N days) → uncollected
```

A single line status:
```
pending → issued_to_vendor → in_stitching → stitching_done → qc_passed → ready → delivered
                                                                  └─ (fail QC) → rework
```

The order-level status is automatically derived from the aggregate of line statuses.

---

## Business Rules

1. **Advance minimum.** At booking, the advance must be ≥ max(`stitching.advance_min_percent` × total, `stitching.advance_min_amount`). Both settings are in Shop Settings. The cashier is blocked if this isn't met.
2. **Measurement snapshot.** At order creation, the selected measurement set values are copied into `measurement_snapshot` on the order line. The FK is kept for traceability, but the order is not affected by future measurement updates.
3. **Fabric from shop.** If `fabric_source = shop`, the cashier selects the roll and enters the fabric length. The roll's `remaining_length` is decremented immediately on order save (not when it goes to the vendor). A `roll_cuts` record is created with `cut_type = stitching_order`.
4. **Customer's own fabric.** If `fabric_source = customer_own`, no roll is decremented. A description field captures the fabric details. The QR tag is still printed and attached to the fabric.
5. **QR tag.** When `stitching.qr_tag_required = true`, the order cannot be saved without printing at least one QR tag per line. The QR tag is printed from Module 20 and physically attached to the fabric.
6. **Urgent orders.** If `is_urgent = true`, `urgency_surcharge_amount` is calculated as `stitching.default_urgency_surcharge_percent × subtotal`. Shown as a separate line on the bill.
7. **Partial delivery.** Individual lines can be marked `delivered` even if others are not ready. The order status becomes `partially_delivered` until all lines are delivered.
8. **Balance collection at delivery.** When a customer collects an order (or a batch of lines), the remaining `balance_due` is collected. A `stitching_order_payments` record is created.
9. **Uncollected orders.** If an order status is `ready` and the `promised_delivery_date + uncollected_days` (from settings) has passed, the status transitions to `uncollected` automatically (scheduled job). A WhatsApp/SMS reminder is sent (Module 37).
10. **Free alteration window.** After delivery, alterations within `stitching.alteration_free_days` are free (Module 26).

---

## Workflows

### Booking a stitching order

1. Staff opens new stitching order. Selects customer (phone lookup).
2. Adds lines: for each garment, selects family member, garment template.
3. System auto-loads the latest measurement set for (member, garment type). Staff can change to a historical version.
4. Staff enters style notes (collar, pocket, embroidery, etc.).
5. Staff selects fabric source (shop or customer's own). If shop, selects roll and length.
6. System computes stitching charges (from rate card or manual entry), adds urgency surcharge if applicable, applies tax.
7. Staff enters advance amount and payment method.
8. System validates: advance ≥ minimum; roll length ≥ required fabric; QR tag printed (if required).
9. Order saved. Bill printed. QR tags printed and handed to staff to attach to fabric. WhatsApp confirmation sent to customer.

### Marking a line as delivered

1. Staff scans the QR tag on the garment, or opens the order by number.
2. Confirms the line is being handed to the customer.
3. Collects balance payment if any.
4. Line status → `delivered`. Order-level status recalculated.
5. If `balance_due = 0`: delivery receipt printed.
6. WhatsApp notification sent to customer: "Your order is delivered."

---

## Permissions

| Permission | Who can |
|------------|---------|
| `stitching_orders.create` | Cashier, Manager, Admin |
| `stitching_orders.update` | Cashier (own session), Manager, Admin |
| `stitching_orders.cancel` | Manager, Admin |
| `stitching_orders.deliver` | Cashier, Manager, Admin |
| `stitching_orders.view` | All authenticated |
| `stitching_orders.collect_payment` | Cashier, Manager, Admin |

---

## Dependencies

- Module 05 (Tax Engine): tax on stitching charges.
- Module 06 (Fiscal): fiscal submission.
- Module 10 (Customers): customer and family member lookup.
- Module 11 (Measurements): measurement selection and snapshot.
- Module 16 (Roll Inventory): fabric cut from roll.
- Module 20 (Barcode & Labels): QR tag printing.
- Module 25 (Vendor Job Management): vendor assignment for each line.
- Module 26 (Alterations): free alteration window.
- Module 29 (Calendar): delivery date availability check.
- Module 31 (Customer Ledger): balance_due creates a receivable.
- Module 37 (Notifications): booking confirmation, ready, uncollected reminders.

---

## Edge Cases

- **Customer cancels after fabric is cut.** If fabric was cut from a shop roll, the cut is already recorded. The fabric becomes a remnant or is written off. A cancellation fee may apply (configurable, out of MVP scope — manual for now).
- **Vendor assigned, then changed.** When a vendor is changed after job issue, the original job slip is cancelled (Module 25) and a new one is created.
- **Two lines for the same family member.** Allowed — e.g. a customer orders two dishdashas for the same son (one for Eid, one for daily wear).
- **Promised date falls on a holiday.** The calendar module checks this and warns at booking. The cashier can override.
- **Order delivered, then alteration requested after the free window.** A new alteration order is created (Module 26) and charged separately.

---

## Out of Scope (MVP)

- Design mood board / image upload per order line (Module 24 is a separate gallery).
- Automated stitching charge calculation from a service price list (MVP is manual entry; a rate card per garment type is a later feature).
- Customer self-booking via ecommerce (Module 39).
- Partial advance refund on cancellation.
