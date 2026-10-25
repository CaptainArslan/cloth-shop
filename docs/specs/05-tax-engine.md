# Module 05 — Tax Engine

**Status:** Draft  
**Phase:** Foundation  
**Last updated:** 2026-10-25

---

## Purpose & Scope

A pluggable, configuration-driven tax engine that computes tax on any sale line without hard-coding country rules into business logic. Tax rates, applicability rules, and inclusive/exclusive pricing are all data, not code. The engine is consumed by POS (Module 21), stitching orders (Module 23), ironing (Module 27), and any other billable module.

Scope: tax categories, rate table, per-line calculation, rounding, inclusive vs exclusive pricing, compound taxes, and tax summary on receipts/invoices.

---

## Tax Concepts Used in Target Countries

### Oman

- **VAT**: 5% flat rate. Applies to most goods and services.
- Introduced January 2021. Standard-rated, zero-rated, and exempt categories exist.
- For a cloth shop: fabric goods = standard-rated (5%); stitching services = standard-rated (5%).
- Invoices to VAT-registered businesses must include seller TRN and buyer TRN.
- Prices are typically **inclusive** of VAT at retail level.

### Pakistan

- **Federal Excise Duty / GST on goods**: 18% standard rate (may change; must be config).
- **Provincial Sales Tax on Services (PST)**: 13–16% depending on province.
- Cloth shops: fabric sales = FED/GST; stitching = PST on services.
- POS receipt must show tax separately (FBR requirement).
- Prices typically **exclusive** of tax at POS level.

---

## Entities & Key Fields

### `tax_categories`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `code` | string | Unique per tenant; e.g. `goods_standard`, `services_standard`, `exempt` |
| `name` | JSON | {en, ar, ur} |
| `description` | text | nullable |

### `tax_rates`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `tax_category_id` | FK | |
| `name` | JSON | e.g. {en: "VAT 5%", ar: "ضريبة القيمة المضافة 5٪"} |
| `code` | string | e.g. `VAT_5`, `GST_18`, `PST_13` |
| `rate` | decimal(8,4) | Percentage: 5.0000 = 5% |
| `is_compound` | boolean | If true, applied on (subtotal + prior taxes), not on subtotal |
| `priority` | tinyint | Order in which compound taxes stack |
| `tax_type` | enum | `goods` \| `services` \| `both` |
| `inclusive` | boolean | True = tax is embedded in the price; false = tax added on top |
| `effective_from` | date | |
| `effective_to` | date | nullable |
| `created_at` / `updated_at` | timestamps | |

### `product_tax_category` (pivot)

Maps each product or service type to its tax category. Products (Module 15), stitching services, ironing services each declare their tax category.

| Column | Type | Notes |
|--------|------|-------|
| `taxable_type` | string | Polymorphic: Product, StitchingOrderLine, IroningItem |
| `taxable_id` | ULID | |
| `tax_category_id` | FK | |

### `tax_line` (embedded in every billable document)

Not a standalone table — serialised as JSON into the document (sale, order, etc.) and also stored in a `tax_lines` child table for reporting.

| Column | Type | Notes |
|--------|------|-------|
| `document_id` | FK | Polymorphic |
| `document_type` | string | |
| `tax_rate_id` | FK | Rate snapshotted at time of sale |
| `tax_rate_snapshot` | JSON | Full rate record copied at time of sale |
| `taxable_amount` | bigint | Minor units; the amount the tax was applied to |
| `tax_amount` | bigint | Minor units; computed tax |
| `currency_code` | char(3) | |

---

## Business Rules

1. **Rate lookup.** The engine resolves the applicable rate(s) for a line by: taxable_type → tax_category → active tax_rates (where `effective_from <= sale_date AND (effective_to IS NULL OR effective_to >= sale_date)`).
2. **Rate snapshot.** The computed `tax_rate_snapshot` is written into the line at save time and never recalculated. Historical documents always show the rate that was in effect.
3. **Inclusive pricing.**
   - Display price includes tax.
   - Tax extracted: `tax_amount = round(price * rate / (1 + rate))`.
   - Net amount: `net = price - tax_amount`.
4. **Exclusive pricing.**
   - Display price excludes tax.
   - Tax added: `tax_amount = round(price * rate)`.
   - Total: `total = price + tax_amount`.
5. **Compound taxes (Pakistan example).**
   - GST applied first on subtotal.
   - PST applied on (subtotal + GST) if `is_compound = true` and `priority` is higher.
   - The engine iterates rates in `priority` order.
6. **Rounding per line, not per document.** Each line is rounded independently (half-up). No document-level rounding adjustment. If regulatory rounding rules differ (e.g. FBR), the fiscal provider handles the discrepancy.
7. **Exempt lines.** Tax category `exempt` produces a `tax_amount` of zero. The line still appears on the tax summary as exempt.
8. **Zero-rated lines.** A zero-rated line uses a rate of 0.0000 and is listed separately on the tax summary (different from exempt for VAT return purposes).
9. **Mixed document.** A single bill may have lines with different tax categories (e.g. fabric at 5% VAT and stitching at 5% VAT in Oman — same rate but tracked separately for fiscal reporting).
10. **Tax-exempt customers.** A customer marked `tax_exempt = true` (e.g. a diplomatic mission) causes the engine to skip all tax calculation and record tax as zero on all lines.

---

## Calculation Algorithm

```
For each billable line:
    1. Resolve taxable_type → tax_category_id → tax_rates[]  (active on document date)
    2. Sort tax_rates by priority ASC
    3. running_base = line.unit_price_amount * line.quantity
    4. For each rate in tax_rates:
        a. If rate.inclusive:
             tax = round(running_base * rate.rate / (1 + rate.rate))
             net = running_base - tax
           Else:
             tax = round(running_base * rate.rate)
        b. If rate.is_compound: running_base += tax  (for next rate in loop)
        c. Append TaxLine(rate, taxable_amount=running_base_before_this_rate, tax_amount=tax)
    5. line.tax_total = sum(tax_lines[].tax_amount)
    6. line.total = unit_price_amount * quantity + (exclusive taxes) OR unit_price_amount * quantity (inclusive)

Document total:
    subtotal = sum(line.net_amount)
    tax_by_rate = group tax_lines by tax_rate_id, sum tax_amount
    grand_total = subtotal + sum(exclusive tax_totals)
```

---

## Receipt / Invoice Display

Tax summary block on every receipt:

```
Subtotal (excl. tax)      OMR 10.000
VAT 5%                    OMR  0.500
─────────────────────────────────────
Total                     OMR 10.500
```

For inclusive pricing:

```
Total (incl. VAT 5%)      OMR 10.500
VAT included              OMR  0.476
```

---

## Workflows

### Adding a new tax rate (e.g. VAT increase)

1. Tenant admin creates a new `tax_rates` record with the new `rate` and `effective_from` date.
2. Sets `effective_to` on the old rate to one day before `effective_from`.
3. All new documents from that date use the new rate automatically.
4. Historical documents retain their snapshotted rate.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `tax_categories.manage` | Tenant admin |
| `tax_rates.manage` | Tenant admin |
| `tax_rates.view` | All authenticated users |

---

## Dependencies

- Module 02 (Country Packs): seeds initial tax rates from the pack.
- Module 04 (Currency): all tax amounts are in minor units.
- Module 06 (Fiscal Integration): the fiscal provider may impose additional rounding or formatting rules on top of the tax engine's output.
- Module 08 (Audit Log): rate changes audited.
- Module 21 (POS), 23 (Stitching Orders), 27 (Ironing): all consume the tax engine.

---

## Edge Cases

- **No active rate found for a line's tax category.** The system should warn (not silently zero out). Either the category is genuinely exempt, or a rate record is missing. Block the sale with an error message naming the missing category.
- **Rate change mid-day.** If `effective_from` = today, all new documents from the moment of creation use the new rate. Documents saved before the rate was added (same day) retain their snapshot.
- **Negative lines (returns).** Tax is calculated symmetrically — a return line produces a negative tax amount.
- **Discount before or after tax.**
   - Oman: discount applies to the inclusive price first, then tax is extracted from the discounted price.
   - Pakistan: discount applies to the pre-tax amount; tax is then applied to the discounted amount.
   - This behaviour is a country pack setting: `discount_applies_before_tax: true/false`.
- **Tax-exempt customer overrides.** If a customer is marked exempt, the engine skips all tax rates regardless of the product's tax category.

---

## Out of Scope (MVP)

- VAT return generation / submission (handled in Module 06 Fiscal Integration).
- Withholding tax.
- Import duties / customs.
- Tax grouping by invoice for B2B VAT invoices (TRN display is handled in the fiscal module).
- Automated tax filing.
