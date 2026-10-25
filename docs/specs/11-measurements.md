# Module 11 — Measurements (Naap)

**Status:** Draft  
**Phase:** Parties  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Records, versions, and retrieves body measurements (naap) for each family member. Measurements are linked to configurable garment templates — a dishdasha has different measurement fields than a shalwar kameez. All versions are kept; measurements are never overwritten. Special customer preferences ("fit like sample") are also tracked here.

Scope: garment templates, measurement fields, versioned measurement history, "fit like sample" references, and preferred style notes.

---

## Core Concepts

- **Garment template**: defines which measurement fields exist for a garment type (e.g. a Dishdasha template has Tull, Seat, Chest, Sleeve, Collar, etc.).
- **Measurement set**: one recorded instance of a customer's measurements for a specific garment template, at a specific date. Each set is a snapshot in time.
- **Versioning**: when measurements are updated, the old set is retained with its date. The latest version is the one used by default when placing a new order, but staff can select any historical version ("same as last Ramadan").
- **"Fit like sample"**: customer brings a garment they like. Staff notes which measurements match it and tags this set as a reference. A new order can say "fit like sample from 2024-03-15".

---

## Entities & Key Fields

### `garment_templates`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | nullable for system templates; populated for tenant custom templates |
| `name` | JSON | Translatable {en, ar, ur} |
| `code` | string | e.g. `dishdasha`, `shalwar_kameez`, `kurta`, `abaya` |
| `gender` | enum | `male` \| `female` \| `unspecified` \| `any` |
| `country_code` | char(2) | nullable; which country this template is most relevant for |
| `is_system` | boolean | System templates cannot be deleted |
| `active` | boolean | |
| `sort_order` | tinyint | Display order in dropdowns |

### `garment_template_fields`

Defines the fields (measurement points) for each template.

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `garment_template_id` | FK | |
| `name` | JSON | Translatable label, e.g. {en: "Length", ar: "الطول"} |
| `code` | string | e.g. `tull`, `seat`, `chest`, `sleeve`, `collar` |
| `unit` | enum | `cm` \| `inch` — per tenant setting |
| `input_type` | enum | `number` \| `select` \| `text` |
| `options` | JSON | nullable; for `select` type: [{value, label_translations}] |
| `is_required` | boolean | |
| `sort_order` | tinyint | |
| `notes_hint` | JSON | nullable; placeholder hint text, translatable |

### `customer_measurements`

One record per measurement session per family member per garment template.

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `customer_family_member_id` | FK | Which person these measurements belong to |
| `garment_template_id` | FK | |
| `version` | tinyint | Auto-incremented per (family_member, template). 1 = first ever. |
| `is_current` | boolean | True on the latest version; false on superseded ones |
| `recorded_by` | FK → users | Staff who took the measurements |
| `recorded_at` | timestamp | When measurements were taken (not necessarily today) |
| `is_sample_reference` | boolean | True if this set was taken from a sample garment the customer brought |
| `sample_notes` | text | nullable; describe the sample (colour, occasion, what they liked) |
| `values` | JSON | `{field_code: value}` e.g. `{"tull": 56, "chest": 42, "sleeve": 25}` |
| `general_notes` | text | nullable; freeform notes from the tailor |
| `created_at` | timestamp | |

> `values` is stored as JSON for flexibility. The field codes are validated against the garment template's fields at save time. Unused fields are omitted; extra fields are rejected.

### `stitching_order_measurement_snapshot`

When a stitching order is created, the selected measurement set is snapshotted onto the order (not just FK'd). This ensures the order retains the measurements as they were at the time of booking, even if the customer's measurements are updated later.

(Defined fully in Module 23 — Stitching Orders, but the snapshot source is this module.)

---

## Built-in Garment Templates

### Oman Pack

| Template | Code | Key fields |
|----------|------|-----------|
| Dishdasha | `dishdasha` | Tull (length), Chest, Seat, Shoulder, Sleeve, Collar, Front opening, Cuff style |
| Abaya | `abaya` | Length, Chest, Shoulder, Sleeve, Neck |
| Bisht (cloak) | `bisht` | Length, Chest, Shoulder |
| Kumma (cap) | `kumma` | Head circumference |
| Sirwal (underpants) | `sirwal` | Waist, Hip, Length |

### Pakistan Pack

| Template | Code | Key fields |
|----------|------|-----------|
| Shalwar Kameez | `shalwar_kameez` | Qad (length), Chest, Waist, Hip, Sleeve, Collar, Shalwar length, Pacha |
| Kurta | `kurta` | Qad, Chest, Shoulder, Sleeve, Collar |
| Pant-Shirt | `pant_shirt` | Chest, Shoulder, Sleeve, Waist, Hip, Inseam, Outseam |
| Waistcoat | `waistcoat` | Chest, Length, Shoulder |
| Sherwani | `sherwani` | Same as Shalwar Kameez + Collar style |

---

## Business Rules

1. **Never overwrite.** Saving updated measurements always creates a new version (increments `version`, sets new record's `is_current = true`, sets old record's `is_current = false`). Old versions are permanent.
2. **Version numbering per (family_member, template).** Versions are independent across garment types — a family member may be on version 5 for dishdasha and version 1 for bisht.
3. **Measurement unit consistency.** Each tenant chooses `cm` or `inch` as their default unit (in Shop Settings). All measurements are stored in that unit. If the unit is ever changed (uncommon), a migration recalculates all stored values.
4. **Required fields enforced.** Fields marked `is_required = true` must have a non-null, non-empty value in `values` before the measurement set can be saved.
5. **Template field validation.** `values` keys must match the template's `garment_template_fields.code` list. Unknown codes are rejected. Missing optional fields are stored as omitted from the JSON (not as null entries).
6. **Sample reference.** At most one measurement set per (family_member, template) can be marked `is_sample_reference = true` at a time. (This is a best practice, not a hard DB constraint — it can change.)
7. **Order measurement snapshot.** When an order line is saved, `customer_measurements.values` is copied into the order line. The FK is also kept for traceability, but the order line is not affected by future measurement changes.

---

## Workflows

### Taking measurements (new customer or new garment type)

1. Staff opens customer profile → family member → Measurements tab.
2. Selects garment template.
3. If no previous measurements exist for this template: blank form is shown.
4. Staff fills in fields and saves.
5. Version 1 is created, `is_current = true`.

### Updating measurements

1. Same flow as above, but existing measurements are pre-filled.
2. Staff edits values and saves.
3. New version created; old version retained.

### Viewing measurement history

Staff can see all versions in a table: version number, date, recorded by, values. Clicking any version shows the full detail. An order can reference a specific version using the "fit like same as version X" option.

### Placing a stitching order with existing measurements

1. On the order form, staff selects the garment type.
2. The system auto-loads the latest measurement set for that (family member, garment template).
3. Staff can override to a historical version if the customer requests.
4. Measurements are snapshotted onto the order line.

### "Fit like sample"

1. Customer brings a garment.
2. Staff records measurements from the sample garment.
3. Marks the set as `is_sample_reference = true` and adds `sample_notes`.
4. When placing an order, staff selects "Fit like sample from [date]" and the order references that snapshot.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `measurements.view` | Cashier, Manager, Admin |
| `measurements.create` | Cashier, Manager, Admin |
| `measurements.edit_current` | Cashier (creates new version; cannot delete old versions) |
| `garment_templates.manage` | Admin |

---

## Dependencies

- Module 10 (Customers): measurements belong to `customer_family_members`.
- Module 02 (Country Packs): garment templates seeded from pack.
- Module 08 (Audit Log): new measurement versions logged.
- Module 23 (Stitching Orders): consumes measurement sets; stores snapshots.

---

## Edge Cases

- **Family member with measurements for multiple garment types.** Each template is independent. A member can have measurements for Dishdasha (v3) and Bisht (v1) simultaneously.
- **Garment template deactivated after measurements recorded.** Old measurement sets remain valid and accessible. The template is only hidden from new-measurement dropdowns.
- **Unit change after measurements recorded.** If the tenant switches from `inch` to `cm`, existing stored values are in inches. Either convert all values (migration with a flag), or store the unit alongside each measurement set (safer). Recommended: add `unit` column to `customer_measurements` so historical records carry their original unit.
- **Measurement taken in advance (before cutting).** Measurements can be recorded at any time, independent of any order. They become a permanent profile asset.
- **Duplicate version on concurrent save.** Two staff members save measurements for the same (family_member, template) at the same time. Use a DB transaction with a row lock on the existing `is_current = true` record to guarantee version number uniqueness.

---

## Out of Scope (MVP)

- 3D body scanning / digital measurement import.
- Size chart / standard sizing (S/M/L) as an alternative to naap — deferred.
- Customer self-entry of measurements (ecommerce feature, later phase).
- Measurement templates for non-clothing items (curtains, bedcovers — possible future extension).
