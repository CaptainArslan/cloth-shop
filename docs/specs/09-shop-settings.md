# Module 09 — Shop Settings

**Status:** Draft  
**Phase:** Foundation  
**Last updated:** 2026-10-25

---

## Purpose & Scope

A central, structured configuration store for all tenant- and branch-level operational settings. Settings are typed, validated, audited, and overridable at different scopes (system defaults → tenant → branch). This module defines the settings schema and the admin UI for managing them.

Scope: invoice series configuration, receipt layout, booking cutoffs, operational hours, notification defaults, discount limits, and all other runtime configuration that is too dynamic to put in `.env` or config files.

---

## Settings Architecture

### Scope hierarchy

```
System defaults (code)
    └── Tenant settings  (tenant_settings table)
            └── Branch settings  (branch_settings table)
```

When the application reads a setting, it checks branch first, then tenant, then the system default. This allows a chain of overrides without duplicating every key at every level.

### Setting types

| Type | Storage | Example |
|------|---------|---------|
| `string` | text | `invoice.prefix = "INV"` |
| `integer` | JSON integer | `stitching.advance_min_percent = 30` |
| `decimal` | JSON string (to avoid float) | `discount.max_percent = "15.00"` |
| `boolean` | JSON `true`/`false` | `pos.require_customer = false` |
| `json` | JSON object/array | `receipt.footer_lines = ["Thank you", ""]` |
| `date` / `time` | ISO string | `shop.open_time = "09:00"` |
| `enum` | string | `invoice.number_format = "sequential"` |

### `tenant_settings` / `branch_settings` table

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `tenant_id` | FK | |
| `branch_id` | FK | null for tenant_settings table; required for branch_settings |
| `key` | string | Dot-namespaced |
| `value` | text | JSON-encoded; null = use parent scope default |
| `updated_at` | timestamp | |
| `updated_by` | FK → users | |

---

## Settings Catalogue

### Invoice & Receipt

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `invoice.prefix` | string | "INV" | Tenant-level; e.g. "RW" for Ruwi branch |
| `invoice.next_number` | integer | 1 | Auto-incremented per tenant+prefix |
| `invoice.number_format` | enum | `sequential` | `sequential` \| `yymm-sequential` |
| `invoice.stitching_prefix` | string | "ST" | Separate series for stitching orders |
| `invoice.ironing_prefix` | string | "IR" | |
| `receipt.show_logo` | boolean | true | |
| `receipt.footer_lines` | json | [] | Up to 3 lines of custom footer text |
| `receipt.show_tax_breakdown` | boolean | true | |
| `receipt.show_barcode` | boolean | true | |
| `receipt.copies` | integer | 1 | Default number of receipt copies |

### POS

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `pos.require_customer` | boolean | false | If true, sale cannot proceed without selecting a customer |
| `pos.allow_credit_sale` | boolean | true | Allow sale without full payment |
| `pos.credit_limit_enforced` | boolean | true | Block sale if customer exceeds credit limit |
| `pos.min_advance_cash_percent` | decimal | "0.00" | Minimum cash advance percentage |
| `pos.fiscal_block_receipt` | boolean | false | Seeded from country pack / fiscal provider |

### Discounts

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `discount.cashier_max_percent` | decimal | "10.00" | Max discount cashier can apply without override |
| `discount.manager_max_percent` | decimal | "30.00" | Max discount with manager PIN |
| `discount.require_reason` | boolean | false | Whether a reason is required for any discount |

### Stitching Orders

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `stitching.advance_min_percent` | decimal | "30.00" | Minimum advance required to accept an order |
| `stitching.advance_min_amount` | integer | 0 | In minor units; whichever is higher wins |
| `stitching.default_urgency_surcharge_percent` | decimal | "20.00" | |
| `stitching.qr_tag_required` | boolean | true | Whether a QR tag must be printed before saving |
| `stitching.alteration_free_days` | integer | 7 | Days after delivery within which alterations are free |

### Calendar & Booking

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `calendar.eid_cutoff_days_before` | integer | 7 | Days before Eid after which new orders are not accepted |
| `calendar.max_orders_per_day` | integer | 50 | 0 = unlimited |
| `calendar.show_hijri` | boolean | true | Show Hijri date alongside Gregorian |
| `shop.open_time` | time | "09:00" | |
| `shop.close_time` | time | "22:00" | |
| `shop.weekend_days` | json | Seeded from country pack | Override per branch |

### Notifications

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `notifications.channel` | enum | `whatsapp` | `whatsapp` \| `sms` \| `both` \| `none` |
| `notifications.order_booked` | boolean | true | Send when stitching order is created |
| `notifications.order_ready` | boolean | true | Send when order status = ready |
| `notifications.uncollected_days` | integer | 7 | Days after ready before uncollected reminder |
| `notifications.uncollected_repeat_days` | integer | 3 | Repeat reminder interval |

### Cash & Finance

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `cash.require_closing_note` | boolean | false | |
| `cash.petty_cash_float` | integer | 0 | Expected float in minor units |
| `cash.day_end_time` | time | "23:00" | Auto-close cash session at this time if not manually closed |

---

## Business Rules

1. **Type validation.** Every setting has a declared type and optional validation rules. Saving an invalid value (e.g. a string for an integer key) is rejected with a validation error.
2. **Audit on change.** Every setting change is written to the audit log with old and new values.
3. **Branch overrides.** A branch setting of `null` means "use tenant default". Setting it to any valid value overrides the tenant default for that branch only.
4. **Invoice number atomicity.** `invoice.next_number` increments must be atomic (DB-level locking or `INCREMENT` in the sequence table) to prevent duplicate invoice numbers in concurrent POS usage.
5. **Read-only in certain states.** Some settings cannot be changed if active records depend on them. For example, `invoice.prefix` should not be changed after invoices exist under that prefix (warn and require confirmation).
6. **Setting visibility.** Not all settings are exposed to all roles. Sensitive financial settings (discount limits, credit limit enforcement) are only editable by admin.

---

## Invoice Number Sequence

Invoice numbers follow a format defined by `invoice.number_format`:

- `sequential`: `{prefix}-00001`, `{prefix}-00002`, …
- `yymm-sequential`: `{prefix}-2510-0001` (resets monthly or yearly — configurable)

Sequence is stored in a dedicated `invoice_sequences` table:

| Column | Type | Notes |
|--------|------|-------|
| `tenant_id` | FK | |
| `branch_id` | FK | nullable for tenant-wide sequences |
| `prefix` | string | |
| `period` | string | null for sequential; "2510" for year-month |
| `next_number` | integer | |

Increment uses `SELECT ... FOR UPDATE` or a DB sequence to guarantee uniqueness under concurrent load.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `settings.view` | Manager, Admin |
| `settings.manage` | Admin |
| `settings.manage_financial` | Admin (subset: discount limits, credit) |

---

## Dependencies

- Module 01 (Tenant & Branch): tenant_id and branch_id as scopes.
- Module 02 (Country Packs): initial values seeded from the pack.
- Module 08 (Audit Log): all changes audited.
- All other modules: read settings via the `Settings` facade/service.

---

## Edge Cases

- **Concurrent invoice numbering.** Two cashiers on different terminals submit sales at the same millisecond. The sequence lock must guarantee no duplicate. Use `LOCK IN SHARE MODE` / `FOR UPDATE` on the sequence row, or a DB-native auto-increment per prefix.
- **Setting removed from the catalogue.** Old keys in `tenant_settings` that no longer exist in the catalogue are ignored but not deleted. A migration can clean them up.
- **Branch with no override inherits tenant changes.** If the tenant admin changes a setting and branches don't have their own override, the change takes effect immediately for all branches. This is by design and should be communicated clearly in the UI.
- **Eid cutoff with unannounced date.** If `calendar.eid_cutoff_days_before = 7` but the Eid Gregorian date is not yet confirmed (Module 02 edge case), the cutoff cannot be computed. Show a warning in the calendar; do not block bookings until the date is confirmed.

---

## Out of Scope (MVP)

- UI-driven export/import of settings (backup and restore settings as JSON).
- Environment-specific settings profiles (dev / staging / prod).
- Webhook-triggered setting changes from external systems.
- Per-user setting overrides (user preferences beyond language are in their profile, not here).
