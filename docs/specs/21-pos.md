# Module 21 — POS (Point of Sale)

**Status:** Draft  
**Phase:** Sales & Services  
**Last updated:** 2026-10-25

---

## Purpose & Scope

The primary sales interface at the counter. Handles fabric sales (cut from rolls), fixed packs, accessories, and service charges. Supports split payments (cash + card + mobile money), multi-currency payment, hold bills, returns, day-end cash closing, and fiscal receipt printing. Designed to be offline-capable (Module 41).

Scope: sale creation, line items, payments, receipt, and fiscal submission. Returns and exchanges are in Module 22. Cash closing is in Module 35.

---

## Entities & Key Fields

### `sales`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | |
| `sale_number` | string | From invoice sequence |
| `customer_id` | FK | nullable (walk-in allowed unless pos.require_customer = true) |
| `cashier_id` | FK → users | |
| `status` | enum | `draft` \| `held` \| `completed` \| `voided` |
| `sale_date` | date | |
| `subtotal_amount` | bigint | Sum of net line amounts; minor units |
| `discount_amount` | bigint | Document-level discount; minor units |
| `tax_amount` | bigint | Total tax; minor units |
| `total_amount` | bigint | Grand total; minor units |
| `paid_amount` | bigint | Total collected across all payment lines |
| `change_amount` | bigint | Overpayment returned as change |
| `balance_due` | bigint | `total - paid`; 0 for fully paid; positive for credit sale |
| `currency_code` | char(3) | Sale currency (branch operating currency) |
| `fiscal_status` | enum | From Module 06 |
| `fiscal_number` / `fiscal_qr_payload` | string | From Module 06 |
| `notes` | text | nullable |
| `created_at` / `updated_at` | timestamps | |

### `sale_lines`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `sale_id` | FK | |
| `product_id` | FK | |
| `fabric_roll_id` | FK | nullable; required for sell_by=length products |
| `description` | string | Snapshot of product name at time of sale |
| `quantity` | decimal(10,3) | Metres, packs, or pieces |
| `unit_price_amount` | bigint | Per-unit price at time of sale; minor units |
| `discount_percent` | decimal(5,2) | Line-level discount |
| `discount_amount` | bigint | Computed; minor units |
| `net_amount` | bigint | `unit_price × qty - discount`; minor units |
| `tax_amount` | bigint | Computed by tax engine |
| `total_amount` | bigint | `net + tax` |
| `currency_code` | char(3) | |
| `tax_category_id` | FK | Snapshotted |
| `tax_rate_snapshot` | JSON | Full rate at time of sale |

### `sale_payments`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `sale_id` | FK | |
| `payment_method_id` | FK | |
| `amount` | bigint | Amount paid via this method; minor units |
| `currency_code` | char(3) | Payment currency (may differ from sale currency) |
| `amount_in_sale_currency` | bigint | Converted to sale currency at locked rate |
| `exchange_rate_id` | FK | nullable |
| `reference` | string | nullable; card auth code, transaction ID, cheque number |
| `collected_at` | timestamp | |

---

## Payment Flow

A sale can have multiple payment lines (split payment):
- **Example**: Total OMR 25.000 — Customer pays OMR 20.000 cash + OMR 5.000 by card.
- **Multi-currency**: Customer pays USD 25 for an OMR 9.500 sale. The exchange rate is applied; change (if any) is given in the sale currency.
- **Credit sale**: If `paid_amount < total_amount`, the difference is recorded as `balance_due`. This creates a ledger entry in Module 31 (Customer Ledger). Only allowed if `pos.allow_credit_sale = true` and the customer's credit limit is not exceeded.

---

## Business Rules

1. **Walk-in customer.** If no customer is selected and `pos.require_customer = false`, the sale is saved against the branch's system "Walk-in" customer.
2. **Fabric roll selection.** For `sell_by = length` lines, the cashier must select the specific roll. The roll's `remaining_length` is checked before saving the line.
3. **Discount limits.** Line and document discounts are validated against the cashier's role limit (`discount.cashier_max_percent`). Exceeding it requires a manager PIN, which is recorded in the audit log.
4. **Hold bill.** A sale can be placed on `held` status and retrieved later. Held bills do not decrement inventory; they reserve nothing. Useful when a customer is gathering more items.
5. **Void.** A completed sale can be voided by a manager within the same business day. A void creates a compensating return (Module 22) automatically. After day-end closing (Module 35), voiding requires admin approval and a reason.
6. **Fiscal block.** If `FbrPosProvider` is active and `requiresFiscalBlockBeforePrint() = true`, the receipt cannot be printed until FBR confirms. The POS shows a "Submitting to FBR…" spinner.
7. **Receipt.** Printed immediately after payment for fully paid sales. For credit sales, a "payment acknowledgement" is printed showing the balance due.
8. **Tax-inclusive display.** For Oman (VAT inclusive pricing), prices are shown inclusive; VAT is extracted and shown as a separate line on the receipt.
9. **Offline mode.** Covered in Module 41. The POS must function without internet and sync when connectivity is restored.

---

## Workflows

### Standard fabric sale

1. Cashier opens new sale. Optionally selects customer (phone lookup).
2. Scans or searches for fabric product. Enters length.
3. System shows available rolls; cashier selects one (or scans roll barcode).
4. System checks: remaining_length ≥ quantity. If yes, line is added with computed price and tax.
5. Repeat for additional items.
6. Optional: apply line or document discount.
7. Cashier clicks "Charge". Payment screen opens.
8. Cashier selects payment method(s), enters amounts.
9. System calculates change. Cashier confirms.
10. Sale saved, roll lengths decremented, fiscal submission triggered, receipt printed.

### Credit sale

Steps 1–9 as above. If `paid_amount < total_amount`:
- System checks credit limit (if enforced).
- `balance_due` recorded. Customer ledger entry created (Module 31).
- Receipt printed with balance due clearly shown.

### Multi-currency payment

1. Customer pays in a different currency (e.g. USD in Oman).
2. Cashier selects "Add payment" → "USD Cash".
3. System shows the exchange rate (current rate for USD→OMR).
4. Cashier enters USD amount; system shows OMR equivalent.
5. Change (if any) is shown in OMR.
6. Exchange rate locked on the payment line.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `pos.open` | Cashier, Manager, Admin |
| `pos.hold_bill` | Cashier, Manager, Admin |
| `pos.void` | Manager, Admin |
| `pos.credit_sale` | Cashier (if allowed), Manager, Admin |
| `pos.discount` | Cashier (up to cashier limit), Manager (up to manager limit), Admin |
| `pos.override_discount` | Manager (PIN), Admin |
| `pos.multi_currency` | Cashier, Manager, Admin |

---

## Dependencies

- Module 04 (Currency): exchange rate on multi-currency payments.
- Module 05 (Tax Engine): tax calculation on each line.
- Module 06 (Fiscal Integration): fiscal submission and receipt block.
- Module 10 (Customers): customer lookup, credit limit check.
- Module 15 (Product Catalog): product lookup, pricing.
- Module 16 (Roll Inventory): roll selection and length decrement.
- Module 22 (Returns): voids trigger automatic returns.
- Module 31 (Customer Ledger): credit sale creates a receivable.
- Module 35 (Cash Management): payment amounts feed the cash book.
- Module 41 (Offline POS): offline operation and sync.

---

## Edge Cases

- **Roll becomes exhausted mid-sale draft.** If a cashier has a held bill with a roll selected, and another terminal cuts the remaining length to zero before the held bill is completed, the held bill's roll cut will fail. The system must re-validate roll availability when a held bill is retrieved.
- **Cashier leaves mid-sale.** The held bill is associated with the branch, not the cashier. Any logged-in cashier can retrieve it.
- **Negative payment (change).** Change is always calculated and shown; it is not stored as a separate payment line — it is stored as `change_amount` on the sale.
- **Overpayment (card).** If a card payment is made for more than the total, the excess should ideally be voided with the card machine. The system records `change_amount` and the cashier handles the refund separately.

---

## Out of Scope (MVP)

- Quotations / estimates before sale.
- Layaway (reservation of stock with deposit, collected later).
- Gift cards.
- POS hardware integration beyond printers and barcode scanners (e.g. customer-facing display).
- ecommerce checkout (Module 39).
