# Module 15 — Product Catalog

**Status:** Draft  
**Phase:** Inventory  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Defines all sellable items: fabric (sold by length or as fixed packs), accessories (buttons, thread, lining), and services (stitching, ironing — these are referenced here but priced in their own modules). The catalog is the source of truth for product names, tax categories, unit of measure, and base pricing.

Scope: product categories, products, variants, unit of measure, pricing, and images. Roll-level inventory is in Module 16. Purchases are in Module 17.

---

## Entities & Key Fields

### `product_categories`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `name` | JSON | Translatable |
| `parent_id` | FK → self | nullable; for sub-categories |
| `tax_category_id` | FK | Determines tax rates for products in this category (Module 05) |
| `sort_order` | tinyint | |
| `active` | boolean | |

### `products`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `category_id` | FK | |
| `name` | JSON | Translatable |
| `code` | string(50) | SKU / internal code; unique per tenant |
| `barcode` | string | nullable; EAN-13 or internal barcode |
| `description` | JSON | Translatable; nullable |
| `sell_by` | enum | `length` \| `pack` \| `piece` |
| `unit` | enum | `meter` \| `yard` \| `piece` \| `pack` |
| `brand` | string | nullable; fabric brand name (e.g. Gul Ahmed, Al-Tayeb) |
| `composition` | string | nullable; e.g. "100% Cotton", "65% Polyester 35% Cotton" |
| `width_cm` | decimal(6,2) | nullable; fabric width in cm (relevant for rolls) |
| `tax_category_id` | FK | Overrides category-level tax category if set |
| `image_path` | string | nullable |
| `active` | boolean | |
| `created_at` / `updated_at` | timestamps | |

### `product_prices`

Versioned price table. A product can have different prices per branch, and prices change over time.

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `product_id` | FK | |
| `branch_id` | FK | nullable = all branches |
| `price_amount` | bigint | Minor units; the selling price per unit |
| `currency_code` | char(3) | |
| `price_includes_tax` | boolean | Is the listed price inclusive of tax? |
| `effective_from` | date | |
| `effective_to` | date | nullable |

### `product_packs`

For products with `sell_by = pack` — defines what's in the pack.

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `product_id` | FK | The pack product |
| `component_product_id` | FK → products | The individual item inside the pack |
| `quantity` | decimal(8,3) | e.g. 3.5 meters of fabric component |

---

## Sell-By Types

| Type | Example | Inventory impact |
|------|---------|-----------------|
| `length` | Fabric cut from a roll | Deducts meters from the roll (Module 16) |
| `pack` | 3-piece suit pack (2m shirt + 1.5m trouser cloth) | Deducts component lengths from rolls |
| `piece` | Ready-made kumma, button card | Deducts one unit from piece stock |

---

## Business Rules

1. **Fabric products must link to rolls.** A `sell_by = length` product is always sourced from a roll in Module 16. At time of sale, the system asks "which roll?" to record the cut.
2. **Price lookup order.** For POS pricing: branch-specific active price → tenant-wide active price → product has no price (block sale with error).
3. **Tax category inheritance.** If a product has its own `tax_category_id`, that takes precedence over the category's tax category. This handles exceptions (e.g. a normally taxable category with one zero-rated item).
4. **Pack stock check.** When selling a pack, the system checks that all component rolls have sufficient remaining length. If any component is short, the pack sale is blocked.
5. **Barcode uniqueness.** Barcode must be unique per tenant. Duplicates are flagged at creation.
6. **Brand as attribute.** `brand` is a simple string on the product — not a separate entity in MVP. If brand management (supplier linkage, brand-level reporting) is needed later, it becomes its own table.

---

## Workflows

### Adding a fabric product

1. Admin enters name (in all languages), code, category, `sell_by = length`, unit = meter, brand, composition, width.
2. Sets price(s) — branch-specific or tenant-wide.
3. Assigns tax category (inherited from product category by default; override if needed).
4. Product is available for roll assignment (Module 16) and POS.

### Adding a pack product

1. Admin creates the pack product (`sell_by = pack`).
2. Adds pack components: select each component product and enter quantity per pack.
3. Sets pack price.
4. At sale time, the system checks stock for all components.

### Price change

1. Admin navigates to product → Prices.
2. Enters new price with `effective_from` date.
3. System closes the current price's `effective_to` to one day before.
4. Historical sales retain their snapshotted price.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `products.view` | All authenticated users |
| `products.create` | Manager, Admin |
| `products.update` | Manager, Admin |
| `products.deactivate` | Admin |
| `product_prices.manage` | Manager, Admin |
| `product_categories.manage` | Admin |

---

## Dependencies

- Module 05 (Tax Engine): `tax_category_id` feeds the tax engine.
- Module 16 (Roll Inventory): `sell_by = length` products are fulfilled from rolls.
- Module 20 (Barcode & Labels): barcode field used for label printing.
- Module 21 (POS): product lookup and pricing.

---

## Edge Cases

- **Product deactivated mid-sale.** If a product is deactivated while a draft sale is open on a POS terminal, the terminal still completes the sale (local draft). Subsequent new sales cannot include the deactivated product.
- **No active price for a product.** POS blocks adding that product to a sale; shows a "price not set" error. Admin must set a price before it can be sold.
- **Pack with a deactivated component.** System warns when a component is deactivated; the pack itself should also be deactivated or the component replaced.

---

## Out of Scope (MVP)

- Product variants (e.g. same fabric in different colors/widths as separate variant SKUs) — use separate products for now.
- Supplier price lists linked to products.
- Minimum selling price / margin protection.
- Product image gallery (single image only in MVP).
- Ecommerce product page content (rich descriptions, SEO) — handled in Module 39.
