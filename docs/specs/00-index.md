# Cloth Shop System — Module Spec Index

**Owner:** M Arslan  
**Status:** In progress  
**Last updated:** 2026-10-25

## How to read these specs

Each spec covers: purpose & scope, entities & key fields, business rules, workflows & statuses, permissions, dependencies, edge cases, and MVP scope.

---

## Foundation (must be designed before Phase 1)

| # | Module | Spec | Status |
|---|--------|------|--------|
| 1 | Tenant & Branch Management | [01-tenant-branch.md](01-tenant-branch.md) | Draft |
| 2 | Country Packs & Localization | [02-country-packs.md](02-country-packs.md) | Draft |
| 3 | Multilingual (Arabic / English / Urdu, RTL) | [03-multilingual.md](03-multilingual.md) | Draft |
| 4 | Currency & Exchange Rates | [04-currency-exchange.md](04-currency-exchange.md) | Draft |
| 5 | Tax Engine | [05-tax-engine.md](05-tax-engine.md) | Draft |
| 6 | Fiscal Integration | [06-fiscal-integration.md](06-fiscal-integration.md) | Draft |
| 7 | Users, Roles & Permissions | [07-users-roles-permissions.md](07-users-roles-permissions.md) | Draft |
| 8 | Audit Log | [08-audit-log.md](08-audit-log.md) | Draft |
| 9 | Shop Settings | [09-shop-settings.md](09-shop-settings.md) | Draft |

## Parties

| # | Module | Spec | Status |
|---|--------|------|--------|
| 10 | Customers | [10-customers.md](10-customers.md) | Draft |
| 11 | Measurements | [11-measurements.md](11-measurements.md) | Draft |
| 12 | Suppliers | [12-suppliers.md](12-suppliers.md) | Draft |
| 13 | Vendors — Darzi / Karigar | [13-vendors-darzi.md](13-vendors-darzi.md) | Draft |
| 14 | Employees | [14-employees.md](14-employees.md) | Draft |

## Inventory

| # | Module | Spec | Status |
|---|--------|------|--------|
| 15 | Product Catalog | [15-product-catalog.md](15-product-catalog.md) | Draft |
| 16 | Roll Inventory | [16-roll-inventory.md](16-roll-inventory.md) | Draft |
| 17 | Purchases | [17-purchases.md](17-purchases.md) | Draft |
| 18 | Stock Transfers | [18-stock-transfers.md](18-stock-transfers.md) | Draft |
| 19 | Stock Audit | [19-stock-audit.md](19-stock-audit.md) | Draft |
| 20 | Barcode & Label Printing | [20-barcode-labels.md](20-barcode-labels.md) | Draft |

## Sales & Services

| # | Module | Spec | Status |
|---|--------|------|--------|
| 21 | POS | [21-pos.md](21-pos.md) | Draft |
| 22 | Returns & Exchanges | [22-returns-exchanges.md](22-returns-exchanges.md) | Draft |
| 23 | Stitching Orders | [23-stitching-orders.md](23-stitching-orders.md) | Draft |
| 24 | Dress Design Gallery | [24-dress-design-gallery.md](24-dress-design-gallery.md) | Draft |
| 25 | Vendor Job Management | [25-vendor-job-management.md](25-vendor-job-management.md) | Draft |
| 26 | Alterations & Rework | [26-alterations-rework.md](26-alterations-rework.md) | Draft |
| 27 | Ironing | [27-ironing.md](27-ironing.md) | Draft |
| 28 | Delivery & Pickup | [28-delivery-pickup.md](28-delivery-pickup.md) | Draft |

## Planning

| # | Module | Spec | Status |
|---|--------|------|--------|
| 29 | Unified Calendar | [29-unified-calendar.md](29-unified-calendar.md) | Draft |
| 30 | Capacity & Booking Control | [30-capacity-booking-control.md](30-capacity-booking-control.md) | Draft |

## Finance

| # | Module | Spec | Status |
|---|--------|------|--------|
| 31 | Customer Ledger | [31-customer-ledger.md](31-customer-ledger.md) | Draft |
| 32 | Vendor Ledger | [32-vendor-ledger.md](32-vendor-ledger.md) | Draft |
| 33 | Supplier Ledger | [33-supplier-ledger.md](33-supplier-ledger.md) | Draft |
| 34 | Expenses | [34-expenses.md](34-expenses.md) | Draft |
| 35 | Accounts & Cash Management | [35-cash-management.md](35-cash-management.md) | Draft |
| 36 | Profit & Loss | [36-profit-loss.md](36-profit-loss.md) | Draft |

## Communication

| # | Module | Spec | Status |
|---|--------|------|--------|
| 37 | Notifications | [37-notifications.md](37-notifications.md) | Draft |

## Reports

| # | Module | Spec | Status |
|---|--------|------|--------|
| 38 | Reports & Dashboard | [38-reports-dashboard.md](38-reports-dashboard.md) | Draft |

## Online

| # | Module | Spec | Status |
|---|--------|------|--------|
| 39 | Ecommerce Storefront | [39-ecommerce-storefront.md](39-ecommerce-storefront.md) | Draft |
| 40 | Online Orders & Couriers | [40-online-orders-couriers.md](40-online-orders-couriers.md) | Draft |

## Technical

| # | Module | Spec | Status |
|---|--------|------|--------|
| 41 | Offline POS & Sync | [41-offline-pos-sync.md](41-offline-pos-sync.md) | Draft |
| 42 | Backups & Data Export | [42-backups-data-export.md](42-backups-data-export.md) | Draft |

---

## Open questions (from project brief)

1. Own-shop only, or multi-tenant SaaS to sell to other shops?
2. Main customer base: Omanis (dishdasha) or South Asian expats (shalwar kameez)?
3. "Repairs" — customer garment alterations, or shop equipment repairs?
4. "Brand" — fabric brand attribute, or something else?
5. "Transfers" — stock, cash, or both?
6. Does Oman's Fawtara e-invoicing mandate apply to this shop, and from when?
