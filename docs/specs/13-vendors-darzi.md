# Module 13 — Vendors — Darzi / Karigar

**Status:** Draft  
**Phase:** Parties  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages the external tailors (darzi / karigar) to whom stitching work is outsourced. Each vendor has a rate card (what they charge per garment type), a capacity limit (how many pieces they can handle per day), and a ledger tracking what is owed to them. This module also defines the job-issue and piece-receiving workflow at a high level (the detailed workflow lives in Module 25 — Vendor Job Management).

Scope: vendor master data, rate cards, capacity, and payable balance. Job issue slips, material issue, and QC are in Module 25.

---

## Entities & Key Fields

### `vendors`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `name` | string(255) | |
| `code` | string(20) | Short code; unique per tenant |
| `type` | enum | `individual` \| `workshop` |
| `phone` | string | E.164; primary contact |
| `alternate_phone` | string | nullable |
| `address` | JSON | nullable |
| `specialisations` | JSON | Array of garment template codes they can stitch: `["dishdasha", "bisht"]` |
| `currency_code` | char(3) | Currency in which rates and payments are expressed |
| `payment_terms_days` | tinyint | Net payment days; 0 = on delivery |
| `active` | boolean | |
| `notes` | text | nullable |
| `created_at` / `updated_at` | timestamps | |

### `vendor_rate_cards`

One record per (vendor, garment template). Defines what the shop pays the vendor per piece.

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `vendor_id` | FK | |
| `garment_template_id` | FK | |
| `rate_amount` | bigint | Minor units — cost per piece |
| `currency_code` | char(3) | |
| `effective_from` | date | |
| `effective_to` | date | nullable |
| `notes` | text | nullable; e.g. "Includes embroidery" |

> When a job is issued to a vendor, the rate is looked up from the active rate card and locked onto the job line. Historical job costs are not affected by future rate changes.

### `vendor_capacity`

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `vendor_id` | FK | |
| `branch_id` | FK | Capacity is per branch (a vendor may work for multiple branches) |
| `max_pieces_per_day` | smallint | 0 = unlimited |
| `effective_from` | date | |
| `effective_to` | date | nullable |

---

## Business Rules

1. **Specialisation filter.** When issuing a job for a particular garment type, only vendors whose `specialisations` array includes that garment template code are shown. Issuing to an unspecialised vendor requires an explicit override with a warning.
2. **Rate card required before issuing jobs.** A vendor must have an active rate card for the garment type before a job can be issued. Missing rate cards are surfaced as a configuration warning.
3. **Rate locked on job issue.** When a job is issued (Module 25), the active rate is copied onto the job line. Subsequent rate card changes do not affect issued jobs.
4. **Capacity check.** Before issuing a job with a target-ready date, the system checks the vendor's `max_pieces_per_day` against how many pieces are already assigned to them for that date range. If over capacity, the user is warned (soft block; override allowed).
5. **Payable tracking.** Each received and QC-passed piece generates a payable entry in the Vendor Ledger (Module 32). The payable is in the vendor's `currency_code`.
6. **Deactivation.** A vendor with open jobs (issued but not received) cannot be deactivated without resolving the jobs or reassigning them to another vendor.

---

## Rate Card Example

| Vendor | Garment | Rate (OMR) | Effective |
|--------|---------|------------|-----------|
| Al-Khalid Tailors | Dishdasha | 3.500 | 2024-01-01 |
| Al-Khalid Tailors | Bisht | 6.000 | 2024-01-01 |
| Hameed Karigar | Shalwar Kameez | PKR 800 | 2024-06-01 |

---

## Workflows

### Adding a vendor

1. Admin enters vendor details, specialisations.
2. Sets up rate card entries per garment type.
3. Sets daily capacity per branch.
4. Vendor is available for job issue.

### Rate card update

1. Admin creates a new rate card entry with a future `effective_from`.
2. System closes the existing rate's `effective_to` to one day before.
3. All new jobs from that date use the new rate; existing issued jobs retain old rate.

### Viewing vendor workload

Calendar view (Module 29) shows how many pieces are assigned to each vendor per day. Used to decide whether to issue more work.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `vendors.view` | Manager, Admin |
| `vendors.create` | Manager, Admin |
| `vendors.update` | Manager, Admin |
| `vendors.deactivate` | Admin |
| `vendor_rate_cards.manage` | Admin |
| `vendor_capacity.manage` | Manager, Admin |

---

## Dependencies

- Module 04 (Currency): vendor rates in minor units; may be in a different currency than the shop.
- Module 08 (Audit Log): rate changes audited.
- Module 11 (Measurements / Garment Templates): `specialisations` references garment template codes.
- Module 25 (Vendor Job Management): issues jobs, tracks piece-level status, records receipt and QC.
- Module 29 (Calendar): vendor capacity displayed on the calendar.
- Module 32 (Vendor Ledger): tracks payables per vendor.

---

## Edge Cases

- **Vendor works for multiple branches.** Capacity is tracked per `(vendor, branch)`. A vendor can have 10 pieces/day for Branch A and 5 pieces/day for Branch B independently.
- **Vendor rate in different currency.** If the shop is in Oman but a vendor quotes in PKR, the payable is tracked in PKR in the ledger and converted to OMR for reporting at the exchange rate on the payment date.
- **No rate card for a garment type.** Job issue is blocked for that garment type. Admin must add a rate first.
- **Vendor capacity = 0 (unlimited).** The capacity check is skipped; no warning is shown.

---

## Out of Scope (MVP)

- Vendor self-service portal (submit received jobs, view payables).
- Automated WhatsApp to vendor when a job is issued.
- Vendor performance tracking (on-time delivery rate, QC failure rate) — captured in data but no dashboard in MVP.
- Piece-rate calculation with deductions for QC failures — flagged as desired future feature.
