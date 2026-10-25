# Module 25 — Vendor Job Management

**Status:** Draft  
**Phase:** Sales & Services  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages the flow of stitching work between the shop and the external vendors (darzi/karigar). Covers issuing job slips, tracking material handed to the vendor, piece-by-piece receiving, QC, and rework. Generates the payable entries in the Vendor Ledger (Module 32).

---

## Entities & Key Fields

### `vendor_jobs`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `vendor_id` | FK | |
| `stitching_order_line_id` | FK | One job per order line |
| `job_number` | string | Auto-generated |
| `status` | enum | `issued` \| `in_progress` \| `returned` \| `qc_passed` \| `qc_failed` \| `rework` |
| `issued_date` | date | |
| `expected_return_date` | date | |
| `actual_return_date` | date | nullable |
| `vendor_rate_amount` | bigint | Rate locked from vendor rate card at issue time |
| `rate_currency_code` | char(3) | |
| `issued_by` | FK → users | |
| `received_by` | FK → users | nullable |
| `qc_by` | FK → users | nullable |
| `notes` | text | nullable |

### `vendor_job_materials`

Materials handed to the vendor along with the job (fabric, lining, buttons, thread).

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `vendor_job_id` | FK | |
| `material_type` | enum | `fabric` \| `lining` \| `button` \| `thread` \| `other` |
| `description` | string | e.g. "White cotton lining, 1.5m" |
| `quantity` | decimal(8,3) | |
| `unit` | string | "meter", "pieces", "spool" |
| `product_id` | FK | nullable; if the material is a shop stock item |
| `fabric_roll_id` | FK | nullable; if fabric material, links to the specific roll |

### `vendor_job_qc`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `vendor_job_id` | FK | |
| `inspection_date` | date | |
| `result` | enum | `pass` \| `fail` |
| `defects` | JSON | Array of noted defects: [{type, description}] |
| `rework_instructions` | text | nullable; instructions sent back to vendor |
| `inspected_by` | FK → users | |

---

## Business Rules

1. **One job per order line.** Each `stitching_order_lines` record has at most one active `vendor_jobs` record at a time. If a vendor is changed, the old job is closed with a cancellation reason and a new one is issued.
2. **Rate locked at issue.** The vendor's rate for the garment type is looked up from the active rate card at the time of issue and stored on the job. Future rate changes do not affect issued jobs.
3. **Materials list.** When a job is issued, staff records all materials handed to the vendor. This creates accountability — if the vendor claims fabric is missing, the material list is the reference.
4. **Job slip printing.** When a job is issued, a job slip is printed (Module 20) containing: order number, customer name, garment type, family member, measurement summary, style notes, design reference (if any), material list, expected return date, and vendor copy.
5. **Piece receiving.** When the vendor returns the finished garment, receiving staff scans the QR tag on the garment, confirms it matches the job, and marks it as `returned`.
6. **QC inspection.** After receiving, a QC check is done: pass → job status `qc_passed`, order line status → `qc_passed → ready`. Fail → job status `qc_failed`, rework instructions noted, garment sent back to vendor.
7. **Payable on QC pass.** When a job passes QC, a vendor payable entry is created in Module 32 (Vendor Ledger) for `vendor_rate_amount`.
8. **Rework.** If QC fails, the vendor must rework the garment. The rework is tracked as a continuation of the same job (status = `rework`). There is no additional payable for rework unless a different rate was negotiated (admin notes this in the job record).

---

## Workflow: Job Issue to QC

```
Order line accepted (pending)
    └─ Staff assigns vendor → vendor_job created (issued)
         └─ Staff prints job slip + materials are handed over
              └─ Vendor works...
                   └─ Vendor returns garment
                        └─ Staff scans QR / confirms receipt → status: returned
                             └─ QC inspection
                                  ├─ Pass → qc_passed → order line: qc_passed → payable created
                                  └─ Fail → qc_failed → rework instructions sent
                                               └─ Vendor reworks → back to 'returned'
```

---

## Permissions

| Permission | Who can |
|------------|---------|
| `vendor_jobs.issue` | Manager, Admin |
| `vendor_jobs.receive` | Manager, Admin, Counter Staff |
| `vendor_jobs.qc` | Manager, Admin |
| `vendor_jobs.view` | All authenticated |

---

## Dependencies

- Module 13 (Vendors): vendor rate card lookup.
- Module 23 (Stitching Orders): order line status updated.
- Module 20 (Barcode & Labels): job slip printing.
- Module 24 (Design Gallery): design reference on job slip.
- Module 29 (Calendar): expected return date vs vendor capacity.
- Module 32 (Vendor Ledger): payable created on QC pass.

---

## Out of Scope (MVP)

- Automated WhatsApp reminder to vendor for overdue jobs (data is available; WhatsApp sending is in Module 37 but triggering rules for vendor jobs are deferred).
- QC defect photo upload.
- Piece-rate deduction for QC failures (rework is unpaid by policy, not by system calculation in MVP).
- Vendor self-service (vendor updates their own job status via a portal).
