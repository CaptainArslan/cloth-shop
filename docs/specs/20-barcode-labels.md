# Module 20 — Barcode & Label Printing

**Status:** Draft  
**Phase:** Inventory  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Generates and prints labels for fabric rolls, remnants, stitching order tracking tags (physical QR codes attached to each garment piece), and price tags. Connects to thermal label printers common at retail/warehouse counters.

---

## Label Types

| Label | Used on | Key data printed |
|-------|---------|-----------------|
| **Roll label** | Fabric rolls (on receipt) | Roll number, product name, barcode, length, colour/brand, branch |
| **Remnant label** | Remnant pieces | Roll number, remnant length, price, barcode |
| **Order tracking tag** | Each garment in a stitching order | Order number, customer name, garment type, family member, darzi name, due date, QR code |
| **Price tag** | Shelf display for fabric rolls | Product name, price per metre, brand, composition |
| **Repayment receipt** | Minor (for petty cash use) | Out of scope for this module |

---

## Entities & Key Fields

### `label_templates`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `type` | enum | `roll` \| `remnant` \| `order_tag` \| `price_tag` |
| `name` | string | e.g. "62mm Roll Label" |
| `paper_width_mm` | decimal | |
| `paper_height_mm` | decimal | |
| `template_content` | text | ZPL or ESC/POS template with placeholders |
| `is_default` | boolean | Default template for this type |

### `print_jobs`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `branch_id` | FK | |
| `label_type` | enum | |
| `reference_type` | string | Polymorphic |
| `reference_id` | ULID | |
| `printer_name` | string | Configured printer for this branch |
| `copies` | tinyint | |
| `status` | enum | `queued` \| `sent` \| `failed` |
| `created_by` | FK → users | |
| `created_at` | timestamp | |

---

## Business Rules

1. **Order tracking tag is mandatory (configurable).** If `stitching.qr_tag_required = true` in settings, a stitching order cannot be saved without printing at least one tracking tag per order line. The QR code encodes the order line ID. Scanning the QR at any stage (counter, darzi, delivery) shows the order status.
2. **Roll label printed on receipt.** When a goods receipt is confirmed (Module 17), the system auto-creates a `print_job` for each new roll. The receiving staff prints them immediately and attaches them.
3. **Printer per branch.** Each branch configures a default label printer (IP or USB). The print job is sent to that printer via a browser print dialog (for web) or a local print agent (for offline POS).
4. **Reprint.** Any label can be reprinted from the source record (roll, order line, etc.) by an authorised user.
5. **Label template.** ZPL (Zebra Printer Language) is the primary format for thermal label printers. A fallback PDF label (printable on any printer with scissors) is also supported.

---

## QR Code on Order Tracking Tag

The QR encodes a URL or a compact identifier:
- Format: `{tenant_slug}/order/{order_id}/line/{line_id}` or a short code that resolves in the system.
- Scanning opens the order detail page (for admin users) or a public-facing status page (for ecommerce customers — Module 39).

---

## Workflow

1. Any module that needs a label calls `LabelService::queue($type, $referenceId, $branchId, $copies)`.
2. A `print_jobs` record is created.
3. The print job is dispatched to the branch's configured printer (ZPL over network, or queued for the local print agent).
4. Status updated to `sent` or `failed`.
5. Staff can reprint failed jobs from the print queue UI.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `labels.print` | Cashier, Manager, Admin |
| `labels.reprint` | Manager, Admin |
| `label_templates.manage` | Admin |

---

## Dependencies

- Module 16 (Roll Inventory): roll and remnant labels.
- Module 23 (Stitching Orders): order tracking tags.
- Module 17 (Purchases): auto-print on receipt.

---

## Out of Scope (MVP)

- Cloud print service integration (Google Cloud Print, PrintNode).
- Handheld barcode scanner integration (scanning at counter via USB HID is supported natively by browsers).
- RFID tags.
- Label design UI (templates are configured by developers/admin via ZPL; a visual designer is a later phase).
