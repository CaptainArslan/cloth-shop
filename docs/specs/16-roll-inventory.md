# Module 16 — Roll Inventory

**Status:** Draft  
**Phase:** Inventory  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Tracks physical fabric rolls from the moment they arrive to the last metre sold. Each roll has a unique ID, a remaining length, and a full cut history. Remnants (short pieces cut from a roll) become sellable items in their own right. Wastage is recorded separately.

Scope: roll creation, cuts, remnants, wastage, roll status lifecycle, and roll-level reporting. Purchasing (how rolls are received) is in Module 17. Labels and barcodes are in Module 20.

---

## Core Concept

A **roll** is one physical bolt of fabric. When the shop receives a purchase, each bolt becomes one `fabric_rolls` record. Every time fabric is cut for a sale, a `roll_cuts` record is created and the roll's `remaining_length` is decremented. When a roll reaches a predefined remnant threshold (or is manually flagged), it becomes a **remnant** — a short piece with its own label, priced differently and tracked as a discrete unit.

---

## Entities & Key Fields

### `fabric_rolls`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `branch_id` | FK | Where the roll is physically located |
| `product_id` | FK → products | The fabric product this roll is an instance of |
| `purchase_line_id` | FK | nullable; which purchase order line it came from (Module 17) |
| `roll_number` | string | Human-readable label, unique per tenant; e.g. "RL-2024-0045" |
| `barcode` | string | unique; printed on the roll label |
| `initial_length` | decimal(8,3) | Metres; as received |
| `remaining_length` | decimal(8,3) | Decremented on each cut |
| `unit` | enum | `meter` \| `yard` |
| `cost_per_unit` | bigint | Minor units; landed cost per metre/yard |
| `cost_currency_code` | char(3) | |
| `status` | enum | `in_stock` \| `remnant` \| `exhausted` \| `transferred` \| `written_off` |
| `remnant_threshold` | decimal(5,3) | Length at which the roll auto-transitions to `remnant`; default from settings |
| `location` | string | nullable; shelf/rack code within the branch |
| `notes` | text | nullable |
| `created_at` / `updated_at` | timestamps | |

### `roll_cuts`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `fabric_roll_id` | FK | |
| `cut_length` | decimal(8,3) | Metres cut |
| `cut_type` | enum | `sale` \| `stitching_order` \| `wastage` \| `remnant_split` \| `stock_audit` |
| `reference_type` | string | Polymorphic: SaleLine, StitchingOrderLine, WastageRecord, etc. |
| `reference_id` | ULID | |
| `remaining_after_cut` | decimal(8,3) | Snapshot of remaining length after this cut |
| `cut_by` | FK → users | |
| `cut_at` | timestamp | |
| `notes` | text | nullable |

### `remnants`

A remnant is a roll that has been flagged as too short to sell as a normal roll. It gets its own label and price.

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `fabric_roll_id` | FK | The roll it came from |
| `length` | decimal(8,3) | Length of the remnant at time of flagging |
| `sell_price_amount` | bigint | Minor units; often discounted from normal per-metre price |
| `currency_code` | char(3) | |
| `sold` | boolean | False until a sale is recorded against this remnant |
| `created_at` | timestamp | |

### `wastage_records`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `fabric_roll_id` | FK | |
| `length` | decimal(8,3) | Metres wasted |
| `reason` | string | e.g. "Cutting error", "Defect", "End piece" |
| `recorded_by` | FK → users | |
| `approved_by` | FK → users | nullable |
| `created_at` | timestamp | |

---

## Roll Status Lifecycle

```
Received (in_stock)
    │
    ├─ Cut for sale / order ──► remaining_length decreases
    │
    ├─ remaining_length ≤ remnant_threshold ──► status = remnant
    │       └─ sold as remnant ──► status = exhausted
    │
    ├─ remaining_length = 0 ──► status = exhausted
    │
    ├─ Transferred to another branch ──► status = transferred (roll reappears at new branch)
    │
    └─ Written off (lost, damaged beyond use) ──► status = written_off
```

---

## Business Rules

1. **Remaining length is the ledger.** `remaining_length` is the authoritative stock figure. It is always decreased by a `roll_cuts` record — never updated directly.
2. **Cut must be ≤ remaining.** A cut that would make `remaining_length` negative is rejected. POS and stitching order creation check this before saving.
3. **Wastage requires approval (configurable).** If `inventory.wastage_requires_approval = true` in settings, a wastage record is created with `approved_by = null` and a notification is sent to the manager. The wastage does not decrement the roll until approved.
4. **Remnant threshold.** When a cut leaves `remaining_length ≤ remnant_threshold`, the roll's status automatically changes to `remnant` and a remnant record is created. The default threshold comes from Shop Settings; it can be overridden per roll.
5. **Roll barcode uniqueness.** Each roll has a unique barcode (system-generated or scanned from supplier label). The barcode is printed on the roll's label (Module 20) and scanned at the POS to identify the correct roll for cutting.
6. **Cost per unit.** The cost per metre is set at the time the roll is received (from the purchase line's landed cost). This cost feeds COGS (cost of goods sold) in the P&L module.
7. **Stock audit.** During a stock audit (Module 19), if the counted length differs from `remaining_length`, an audit adjustment `roll_cuts` record is created with `cut_type = stock_audit`.

---

## Workflows

### Receiving a roll (triggered from Module 17 — Purchases)

1. Purchase order line is received.
2. For each roll received, a `fabric_rolls` record is created: product, branch, initial length, cost per unit.
3. Roll label (barcode) is printed (Module 20).
4. Roll is placed on the shelf; location optionally recorded.

### Cutting fabric for a sale (POS)

1. Cashier adds a `sell_by = length` product to the sale.
2. Enters the length required.
3. System shows available rolls for that product at the branch (sorted by `remaining_length` descending by default, or the cashier scans the roll barcode).
4. Cashier selects the roll.
5. On sale confirmation, a `roll_cuts` record is created and `remaining_length` is decremented.
6. If remaining after cut ≤ remnant_threshold → status becomes `remnant`.

### Cutting fabric for a stitching order

Same as POS cut, but `cut_type = stitching_order` and `reference_id` = the stitching order line ID.

### Marking wastage

1. Staff records a wastage event: roll, length, reason.
2. If approval is required, manager reviews and approves.
3. On approval, a `roll_cuts` record with `cut_type = wastage` is created and `remaining_length` decremented.
4. Wastage is reported on the daily/monthly wastage report.

### Selling a remnant

1. Remnant has its own barcode (printed on label when created).
2. Cashier scans the remnant barcode at POS.
3. System recognises it as a remnant; adds it to the sale at the remnant price (as a fixed piece, not a length).
4. `remnants.sold = true`; roll status remains `remnant` (or moves to `exhausted` if `remaining_length = 0`).

---

## Permissions

| Permission | Who can |
|------------|---------|
| `rolls.view` | All authenticated |
| `rolls.receive` | Manager, Admin (triggered from Purchases) |
| `rolls.cut` | Cashier, Manager, Admin (triggered from POS/Orders) |
| `rolls.wastage` | Manager, Admin |
| `rolls.wastage.approve` | Manager, Admin |
| `rolls.write_off` | Admin |
| `rolls.transfer` | Manager, Admin (triggers Module 18) |

---

## Dependencies

- Module 15 (Product Catalog): roll belongs to a product.
- Module 17 (Purchases): rolls are created on purchase receipt.
- Module 18 (Stock Transfers): `transferred` status; a new roll record is created at the destination branch.
- Module 19 (Stock Audit): audit adjustments write `roll_cuts` records.
- Module 20 (Barcode & Labels): roll labels and remnant labels.
- Module 21 (POS): cuts triggered at point of sale.
- Module 23 (Stitching Orders): cuts triggered by stitching order lines.
- Module 36 (P&L): `cost_per_unit` used for COGS calculation.

---

## Edge Cases

- **Cut partially across two rolls.** Not supported. Each cut comes from exactly one roll. If a required length (e.g. 3.2m) exceeds a roll's remaining length (e.g. 2.5m), the cashier must cut from two rolls in two separate line items, or choose a different roll.
- **Roll received with wrong length.** If the actual measured length differs from the purchase order quantity, the `initial_length` on the roll record must be set to the actual measured length and a discrepancy note added to the purchase line.
- **Roll transferred between branches.** The original roll is marked `transferred`. A new `fabric_rolls` record is created at the destination branch with the same `remaining_length` and a link to the original roll via the stock transfer record (Module 18).
- **Concurrent cuts.** Two terminals cut from the same roll simultaneously. The `remaining_length` update must be done inside a DB transaction with row-level locking on the `fabric_rolls` record to prevent over-cutting.

---

## Out of Scope (MVP)

- Roll-level colour tracking (colour is an attribute on the product or a text field on the roll; not a separate entity in MVP).
- Fabric weight tracking (GSM).
- Minimum stock level alerts per product (data exists; alerts are a later feature).
- FIFO/LIFO cost averaging across rolls of the same product (MVP uses per-roll actual cost).
