# Module 27 — Ironing

**Status:** Draft  
**Phase:** Sales & Services  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages the ironing service. Customers drop off garments; the shop records each piece with dress details (type, colour), charges a per-piece fee, and gives a pickup date. The module is designed to be extensible to washing and dry-cleaning later without structural change.

---

## Entities & Key Fields

### `ironing_orders`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `order_number` | string | From invoice sequence (`ironing_prefix`) |
| `customer_id` | FK | nullable (walk-in allowed) |
| `received_date` | date | |
| `promised_pickup_date` | date | |
| `status` | enum | `received` \| `in_progress` \| `done` \| `delivered` \| `uncollected` |
| `total_amount` | bigint | |
| `paid_amount` | bigint | |
| `balance_due` | bigint | |
| `currency_code` | char(3) | |
| `created_by` | FK → users | |

### `ironing_order_lines`

One record per garment piece.

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `ironing_order_id` | FK | |
| `service_type` | enum | `ironing` \| `washing` \| `dry_cleaning` (extensible) |
| `garment_description` | string | e.g. "White Dishdasha", "Blue Shirt" |
| `garment_type` | string | nullable; free text or linked to garment templates |
| `colour` | string | nullable |
| `special_instructions` | text | nullable |
| `unit_price_amount` | bigint | Per-piece charge |
| `status` | enum | `pending` \| `done` |

---

## Business Rules

1. **Per-piece pricing.** Charges are per piece, defined in a service price list (stored in `product_catalog` as `sell_by = piece` products of type "service" — e.g. "Ironing - Dishdasha" at OMR 0.200 per piece). The cashier selects the service type; price is auto-filled.
2. **Token slip.** On receiving, a token slip (small receipt) is printed with order number, piece count, and pickup date. The customer shows this to collect their order.
3. **Payment.** Payment is typically collected at delivery. Advance payment is accepted but not required.
4. **Uncollected.** Same logic as stitching orders: after `uncollected_days` past `promised_pickup_date`, status → `uncollected`. Reminder sent (Module 37).
5. **Service extensibility.** `service_type` on the line is an enum. Adding "washing" or "dry_cleaning" is a code-level addition to the enum, not a schema change.
6. **Tax.** Ironing services are taxable at the services tax rate (Module 05).

---

## Workflow

1. Staff opens new ironing order. Optionally selects customer.
2. Adds lines: one per garment, selects service type, enters description, system fills price.
3. Enters promised pickup date. Token slip printed.
4. When done, marks lines as done; order status → `done`.
5. Customer notified. Customer arrives; payment collected (if not already paid); status → `delivered`.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `ironing_orders.create` | Cashier, Manager, Admin |
| `ironing_orders.update_status` | Cashier, Manager, Admin |
| `ironing_orders.deliver` | Cashier, Manager, Admin |

---

## Dependencies

- Module 05 (Tax Engine): service tax on ironing.
- Module 10 (Customers): customer lookup.
- Module 15 (Product Catalog): service items for ironing pricing.
- Module 29 (Calendar): pickup date availability.
- Module 37 (Notifications): uncollected reminders.

---

## Out of Scope (MVP)

- Washing and dry-cleaning (enum value exists; UI and pricing deferred).
- Home pickup and delivery integration for ironing (Module 40 is for ecommerce orders).
- Iron machine maintenance scheduling.
