# Module 12 — Suppliers

**Status:** Draft  
**Phase:** Parties  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages the businesses and individuals from whom the shop buys fabric, thread, accessories, and other stock. Tracks supplier contact details, payment terms, credit terms, and links to the Supplier Ledger (Module 33) for outstanding payables.

Scope: supplier master data, contact persons, payment terms, currency preference, and integration with purchasing (Module 17) and the supplier ledger (Module 33). Does not include price lists or catalogues (those are in Module 15 — Product Catalog).

---

## Entities & Key Fields

### `suppliers`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `name` | string(255) | Company or individual name |
| `code` | string(20) | Short code for reports and labels; unique per tenant |
| `type` | enum | `company` \| `individual` |
| `phone` | string | E.164 |
| `alternate_phone` | string | nullable |
| `email` | string | nullable |
| `address` | JSON | {street, city, region, country} |
| `country_code` | char(2) | Supplier's country (affects currency and tax) |
| `currency_code` | char(3) | Default invoice currency for this supplier |
| `tax_id` | string | nullable; VAT/TRN/NTN number |
| `payment_terms_days` | tinyint | e.g. 30 = net 30 days |
| `credit_limit` | bigint | Minor units in tenant base currency; 0 = prepayment only |
| `notes` | text | nullable |
| `active` | boolean | |
| `created_by` | FK → users | |
| `created_at` / `updated_at` | timestamps | |

### `supplier_contacts`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `supplier_id` | FK | |
| `name` | string | |
| `role` | string | nullable; e.g. "Sales rep", "Accounts" |
| `phone` | string | nullable |
| `email` | string | nullable |
| `is_primary` | boolean | Primary contact for communications |

---

## Business Rules

1. **Currency on purchase.** When a purchase order is created against a supplier, the supplier's `currency_code` is pre-filled. If the supplier is overseas (e.g. China, India, UAE), purchases are in that currency and the supplier ledger tracks the foreign-currency payable.
2. **Payment terms.** `payment_terms_days` seeds the due date on each purchase invoice: `due_date = invoice_date + payment_terms_days`. This can be overridden per purchase.
3. **Credit limit.** If the outstanding balance in the Supplier Ledger (Module 33) exceeds `credit_limit`, a warning is shown when creating a new purchase. (Soft block only — purchasing staff can override with a note.)
4. **Deactivation.** A supplier with outstanding payables cannot be fully deactivated; the system warns and requires the balance to be cleared first or an explicit override.
5. **Tax ID on purchase invoices.** The supplier's `tax_id` is printed on purchase invoices and used for VAT input-tax reclaim reporting.

---

## Workflows

### Creating a supplier

1. Purchasing staff enters name, code, country, currency, payment terms.
2. Optionally adds contact persons.
3. Supplier is immediately available for purchase orders.

### Deactivating a supplier

1. System checks for open purchase orders or outstanding ledger balance.
2. If blockers exist, shows a list. Staff resolves them first, or an admin forces deactivation with a note.
3. `active = false`; supplier no longer appears in new purchase dropdowns.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `suppliers.view` | Manager, Admin, Accountant |
| `suppliers.create` | Manager, Admin |
| `suppliers.update` | Manager, Admin |
| `suppliers.deactivate` | Admin |

---

## Dependencies

- Module 04 (Currency): `currency_code` for foreign-currency purchases.
- Module 08 (Audit Log): supplier changes audited.
- Module 17 (Purchases): supplier selected on each purchase order.
- Module 33 (Supplier Ledger): tracks payables and outstanding balance.

---

## Edge Cases

- **Supplier in same country as shop.** `currency_code` matches tenant currency; no exchange rate needed for purchases from this supplier.
- **Supplier changes bank details or contact.** Update on the supplier record; historical purchase records are not affected (they snapshot what was relevant at time of purchase).
- **Multiple branches buying from same supplier.** The supplier is shared at tenant level; ledger balances are per-branch.

---

## Out of Scope (MVP)

- Supplier price lists / catalogues (Module 15).
- Supplier portal / self-service.
- Automated purchase order email to supplier.
- Supplier performance scoring / rating.
