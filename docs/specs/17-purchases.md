# Module 17 — Purchases

**Status:** Draft  
**Phase:** Inventory  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Records all stock purchases from suppliers — fabric rolls and other inventory items. Handles landed cost (freight, customs, local charges added to the per-unit cost), multi-currency invoices, partial deliveries, and purchase returns.

Scope: purchase orders, goods received notes, landed cost, supplier invoices, and purchase returns. Supplier payables are recorded in Module 33 (Supplier Ledger).

---

## Entities & Key Fields

### `purchase_orders`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `branch_id` | FK | Receiving branch |
| `supplier_id` | FK | |
| `po_number` | string | Auto-generated; unique per tenant |
| `status` | enum | `draft` \| `sent` \| `partial` \| `received` \| `cancelled` |
| `order_date` | date | |
| `expected_delivery_date` | date | nullable |
| `currency_code` | char(3) | Supplier invoice currency |
| `exchange_rate_id` | FK | nullable; rate locked when PO is confirmed |
| `notes` | text | nullable |
| `created_by` | FK → users | |
| `created_at` / `updated_at` | timestamps | |

### `purchase_order_lines`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `purchase_order_id` | FK | |
| `product_id` | FK | |
| `quantity` | decimal(10,3) | In product units (metres or pieces) |
| `unit_cost_amount` | bigint | Minor units in PO currency |
| `currency_code` | char(3) | |
| `received_quantity` | decimal(10,3) | Updated on each goods receipt |
| `status` | enum | `pending` \| `partial` \| `received` \| `cancelled` |

### `goods_receipts`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `purchase_order_id` | FK | |
| `branch_id` | FK | |
| `received_by` | FK → users | |
| `received_at` | timestamp | |
| `supplier_invoice_number` | string | nullable |
| `supplier_invoice_date` | date | nullable |
| `notes` | text | nullable |

### `goods_receipt_lines`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `goods_receipt_id` | FK | |
| `purchase_order_line_id` | FK | |
| `received_quantity` | decimal(10,3) | |
| `unit_cost_amount` | bigint | May differ from PO if price changed on delivery |
| `roll_ids` | JSON | Array of `fabric_rolls.id` created from this line |

### `landed_costs`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `goods_receipt_id` | FK | |
| `description` | string | e.g. "Freight", "Customs duty", "Local transport" |
| `amount` | bigint | Minor units in tenant base currency |
| `currency_code` | char(3) | |
| `allocation_method` | enum | `by_quantity` \| `by_value` \| `by_weight` \| `manual` |

Landed costs are allocated across receipt lines using the chosen method and added to each line's `unit_cost_amount` to compute the final landed cost per unit — this is what gets stored as `cost_per_unit` on each `fabric_rolls` record.

### `purchase_returns`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `goods_receipt_id` | FK | |
| `supplier_id` | FK | |
| `return_date` | date | |
| `reason` | text | |
| `status` | enum | `draft` \| `returned` \| `credited` |
| `created_by` | FK → users | |

---

## Business Rules

1. **PO → GRN flow.** Goods are received against a PO line, not free-floating. Every received quantity updates `purchase_order_lines.received_quantity`. When `received_quantity = quantity`, the line status becomes `received`. When all lines are received, the PO status becomes `received`.
2. **Partial deliveries.** If only part of a PO line is received, the PO status is `partial`. Subsequent deliveries create additional `goods_receipts` against the same PO.
3. **Roll creation on receipt.** For fabric products (`sell_by = length`), each received piece (roll) creates one `fabric_rolls` record. The number of rolls and each roll's measured length is entered by the receiving staff. The sum of roll lengths must match `received_quantity` (or the discrepancy is noted).
4. **Landed cost allocation.** Freight and other extra costs are added to the GRN and allocated across lines using the chosen method. The system recalculates `cost_per_unit` for each roll after allocation.
5. **Currency.** If the PO is in a foreign currency, the exchange rate at time of receipt is locked. `cost_per_unit` on the roll is stored in the tenant base currency equivalent.
6. **Supplier invoice number.** Recorded on the GRN for matching against the physical invoice. Used in supplier ledger reconciliation.
7. **Purchase return.** If stock is returned to the supplier, a `purchase_return` record is created. It references the original GRN lines. The returned rolls' status is set to `written_off`. A credit is recorded in the Supplier Ledger (Module 33).
8. **No PO, direct purchase (MVP exception).** For small, unplanned purchases, a simplified flow creates a GRN without a PO (`purchase_order_id = null`). The PO fields are recorded directly on the GRN. This is flagged in reports as "no PO".

---

## Workflows

### Full purchase flow

1. Manager creates a PO: select supplier, add lines (product + quantity + unit cost), set expected delivery date. Status: `draft`.
2. PO is reviewed and optionally sent to the supplier (email or WhatsApp — manual in MVP). Status: `sent`.
3. Goods arrive. Receiving staff opens the PO and creates a GRN: enter actual quantities received per line, enter number of rolls and each roll's measured length for fabric.
4. System creates `fabric_rolls` records from the GRN fabric lines.
5. Receiving staff adds landed costs (freight, etc.) and selects allocation method.
6. System allocates landed costs and sets `cost_per_unit` on each roll.
7. GRN is saved. PO status updated. Supplier ledger entry created (Module 33).
8. Roll labels are printed (Module 20).

### Partial delivery

Steps 3–8 repeated for each delivery. PO status stays `partial` until all lines are fully received.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `purchases.view` | Manager, Admin, Accountant |
| `purchases.create_po` | Manager, Admin |
| `purchases.receive` | Manager, Admin |
| `purchases.add_landed_cost` | Manager, Admin, Accountant |
| `purchases.return` | Manager, Admin |

---

## Dependencies

- Module 12 (Suppliers): supplier on each PO.
- Module 15 (Product Catalog): products on PO lines.
- Module 16 (Roll Inventory): rolls created on GRN.
- Module 04 (Currency): multi-currency POs and exchange rate locking.
- Module 20 (Barcode & Labels): labels printed after receiving.
- Module 33 (Supplier Ledger): payable created on GRN confirmation.

---

## Edge Cases

- **Quantity received > quantity ordered.** System warns (over-delivery). The excess can be accepted (PO line quantity is updated with manager override) or returned.
- **Unit cost on delivery differs from PO.** The receiving staff can enter the actual unit cost on the GRN line. The difference is flagged in a purchase variance report.
- **Rolls don't add up to received quantity.** If the sum of roll lengths doesn't match the GRN received quantity, the system shows the discrepancy and blocks saving until resolved (adjust roll lengths or note the loss as wastage).
- **Currency fluctuation between PO and GRN.** The PO locks an indicative rate; the GRN locks the actual rate on the receipt date. The difference appears as a purchase price variance.

---

## Out of Scope (MVP)

- Purchase order approval workflow (multi-level approval).
- Automated reorder point / replenishment suggestions.
- PO sending via email (manual process in MVP).
- Supplier invoice matching / 3-way matching (PO vs GRN vs invoice).
- Forward contracts or fixed exchange rate bookings.
