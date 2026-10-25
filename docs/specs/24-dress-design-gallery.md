# Module 24 — Dress Design Gallery

**Status:** Draft  
**Phase:** Sales & Services  
**Last updated:** 2026-10-25

---

## Purpose & Scope

A reference library of garment designs, styles, and photos that staff can show to customers when they are choosing what to stitch. Designs can be tagged by garment type, occasion, and style. A customer can point to a design and say "make it like this" — the design is linked to the stitching order line as a reference, and its style notes are pre-populated.

This is a lightweight image gallery with tagging and order linkage — not a 3D configurator.

---

## Entities & Key Fields

### `dress_designs`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `name` | JSON | Translatable |
| `garment_template_id` | FK | |
| `occasion` | string | nullable; e.g. "Eid", "Wedding", "Formal", "Casual" |
| `style_tags` | JSON | Array of free-text tags: ["embroidered", "collar_type_A", "cuff_button"] |
| `description` | JSON | Translatable; nullable |
| `style_notes_template` | text | nullable; pre-filled style notes copied to order lines |
| `image_paths` | JSON | Array of image filenames (up to 10 images per design) |
| `is_public` | boolean | True = visible on ecommerce storefront (Module 39) |
| `active` | boolean | |
| `sort_order` | tinyint | |
| `created_at` / `updated_at` | timestamps | |

---

## Business Rules

1. **Linked to order line as reference only.** When a design is selected for a stitching order line, the design ID is stored and the `style_notes_template` is copied into the order line's `style_notes`. The order line is not dependent on the design record after that — editing the design does not change the order.
2. **Images stored locally.** Images are stored in the application's storage (not a CDN in MVP). Up to 10 images per design.
3. **Public designs.** If `is_public = true`, the design appears on the ecommerce storefront (Module 39) to help customers pick a style before visiting.

---

## Workflow

1. Admin/manager uploads design photos and fills in name, garment type, occasion, style notes.
2. At order booking, staff browses the gallery filtered by garment type.
3. Customer picks a design; staff links it to the order line.
4. `style_notes_template` is copied into the order line's notes. Staff can edit.
5. Design reference is printed on the job issue slip (Module 25) so the vendor knows the style.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `designs.view` | All authenticated |
| `designs.manage` | Manager, Admin |

---

## Dependencies

- Module 11 (Measurements / Garment Templates): `garment_template_id`.
- Module 23 (Stitching Orders): design linked to order lines.
- Module 25 (Vendor Job Management): design reference on job slip.
- Module 39 (Ecommerce): public designs visible on storefront.

---

## Out of Scope (MVP)

- Customer-submitted design photos (ecommerce feature).
- 3D garment visualiser.
- Design variants (showing the same design in different colours).
- Design categories / collections (e.g. "Ramadan 2025 Collection").
