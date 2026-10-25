# Module 22 — Returns & Exchanges

**Status:** Draft  
**Phase:** Sales & Services  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Handles fabric and item returns from customers and exchanges (return + new sale). Restores inventory where possible (fabric returns) or records wastage where it is not. Handles refunds to the original payment method or as store credit.

---

## Entities & Key Fields

### `sale_returns`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `return_number` | string | |
| `original_sale_id` | FK → sales | nullable; for returns without an original sale (rare) |
| `customer_id` | FK | |
| `return_date` | date | |
| `reason` | text | |
| `status` | enum | `draft` \| `completed` \| `voided` |
| `refund_method` | enum | `original_payment` \| `store_credit` \| `cash` |
| `refund_amount` | bigint | Minor units |
| `processed_by` | FK → users | |

### `sale_return_lines`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `sale_return_id` | FK | |
| `original_sale_line_id` | FK | |
| `quantity` | decimal(10,3) | Quantity being returned |
| `unit_price_amount` | bigint | Snapshotted from original line |
| `refund_amount` | bigint | Computed; includes tax reversal |
| `restock` | boolean | Whether the fabric is being put back in stock (vs wastage) |
| `restocked_roll_id` | FK → fabric_rolls | nullable; which roll it was added back to, or a new roll created |

---

## Business Rules

1. **Return must reference original sale.** Returns without a receipt require manager authorisation. The system prompts for the sale number or customer phone to find the original sale.
2. **Return window.** Configurable per setting (`returns.allowed_days`). After this many days from sale, returns require manager approval.
3. **Fabric restock.** Returned fabric can be restocked onto the original roll (if available) or converted to a new remnant record. Not all returned fabric is resellable — staff marks `restock = false` if the fabric is damaged.
4. **Tax reversal.** The refund amount includes the tax component of the returned lines. The tax is reversed and reflected in the fiscal submission (a credit note is sent to FBR if the fiscal provider requires it).
5. **Refund method.** Refund goes back to the original payment method if possible. Cash refunds for card payments require manager approval (and may be restricted by settings). Store credit is added to the customer's credit balance in Module 31.
6. **Exchange.** An exchange is processed as: one return transaction + one new sale. These are linked by an `exchange_id` on both records. The net cash flow is calculated across both.
7. **Void vs return.** A void (same day, by manager) cancels the sale in full. A return (any time, with manager approval if outside window) is a separate transaction that partially or fully reverses the sale.

---

## Workflow

### Customer return

1. Cashier opens "Returns". Enters original sale number or scans receipt barcode.
2. System loads the original sale; staff selects which lines are being returned and quantities.
3. For fabric: staff marks whether the fabric is being restocked.
4. System computes refund amount (with tax).
5. Manager approves if required (outside return window, or cash refund for card payment).
6. Return saved; inventory updated; fiscal credit note submitted if required; refund issued.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `returns.create` | Cashier (within return window), Manager, Admin |
| `returns.approve` | Manager, Admin |
| `returns.without_receipt` | Manager, Admin |

---

## Dependencies

- Module 21 (POS): returns reference original sales.
- Module 16 (Roll Inventory): returned fabric restocked.
- Module 06 (Fiscal): credit note submission.
- Module 31 (Customer Ledger): store credit recorded.

---

## Out of Scope (MVP)

- Return of stitching-order garments (those go through alterations — Module 26).
- Warranty or defect claims against suppliers (those go through purchase returns — Module 17).
