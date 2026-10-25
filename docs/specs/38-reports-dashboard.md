# Module 38 — Reports & Dashboard

**Status:** Draft  
**Phase:** Reports  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Central reporting interface. The dashboard gives a real-time summary for the current day/week. Reports provide detailed data on sales, inventory, stitching operations, finance, and customers. All reports are exportable to CSV/Excel.

---

## Dashboard Widgets

| Widget | Data source | Who sees it |
|--------|------------|------------|
| Today's revenue | POS + stitching payments | Manager, Admin |
| Today's orders (stitching + ironing) | Order counts | Cashier, Manager, Admin |
| Pending deliveries (due today/overdue) | Orders with status = ready + past promised date | All |
| Uncollected orders count | Orders with status = uncollected | Manager, Admin |
| Cash on hand (open register balance) | Current cash session | Cashier, Manager, Admin |
| Top-selling fabrics (this month) | Sale line aggregation | Manager, Admin |
| Vendor jobs overdue | VendorJobs past expected_return_date | Manager, Admin |
| Outstanding customer balances (top 5) | Customer ledger | Manager, Admin |
| Monthly P&L summary | pl_snapshots | Admin, Accountant |

---

## Reports Catalogue

### Sales Reports

| Report | Key filters | Key columns |
|--------|------------|-------------|
| Sales summary | Date range, branch, cashier | Date, invoice #, customer, subtotal, tax, total, payment methods |
| Sales by product | Date range, product, category | Product, metres sold, revenue, COGS, gross margin |
| Sales by cashier | Date range | Cashier, sales count, revenue, discounts given |
| Discount report | Date range | Invoice #, discount amount, discount %, authorised by |
| Credit sales & outstanding | Date range | Customer, invoice #, total, paid, balance, overdue days |

### Inventory Reports

| Report | Key filters | Key columns |
|--------|------------|-------------|
| Stock on hand | Branch, product, category | Roll #, product, remaining metres, cost value |
| Low stock alert | Below threshold | Product, remaining, reorder level |
| Roll cut history | Roll, date range | Roll #, cut date, cut length, reference (sale/order), cut by |
| Wastage report | Date range, branch | Roll #, product, wastage length, reason, approved by |
| Purchase history | Supplier, date range | PO #, GRN #, supplier, product, quantity, cost, landed cost |

### Stitching Reports

| Report | Key filters | Key columns |
|--------|------------|-------------|
| Stitching orders list | Date range, status, vendor | Order #, customer, garment count, promised date, status, balance |
| Orders due this week | Branch | Date, order #, customer, garments, vendor assigned |
| Uncollected orders aging | Branch | Order #, customer, ready date, days overdue, balance due |
| Vendor job status | Vendor, date range | Job #, vendor, garment, order #, issued date, expected return, status |
| Alteration summary | Date range | Alteration #, customer, original order, free/paid, charge |

### Finance Reports

| Report | Key filters | Key columns |
|--------|------------|-------------|
| Day book | Branch, date | Time, transaction type, reference, debit, credit, balance |
| Cash closing summary | Branch, date range | Date, opening float, cash in, cash out, expected, actual, variance |
| Customer statement | Customer, date range | Date, description, debit, credit, running balance |
| Vendor statement | Vendor, date range | Date, job #, payable, payment, balance |
| Supplier statement | Supplier, date range | Date, invoice #, payable, payment, balance |
| P&L | Branch, period | Revenue, COGS, gross profit, expenses, net profit |
| Expense summary | Category, date range | Category, description, amount, paid from |

### Tax Reports

| Report | Key filters | Key columns |
|--------|------------|-------------|
| VAT summary | Date range | Taxable amount, tax rate, tax collected, tax paid on expenses |
| Tax by category | Date range | Tax category, standard/exempt/zero-rated, amounts |

---

## Business Rules

1. **Branch scoping.** All reports default to the user's current branch. Admin can select all branches or a specific branch.
2. **Snapshotted P&L.** The dashboard loads from `pl_snapshots` (nightly job). The full P&L report is computed on demand.
3. **Export.** Every report has a "Export CSV" button. The CSV includes all columns, unformatted (raw numbers, ISO dates) for Excel compatibility.
4. **Date formats.** Reports display dates in the tenant's configured `date_format`. Exported CSV uses ISO 8601 (YYYY-MM-DD).
5. **Amounts.** Reports display amounts formatted per currency (OMR 1.500, PKR 150.00). Exported CSV uses raw minor-unit integers with a currency code column.
6. **Report access is logged.** Report exports are written to the audit log (Module 08): who ran what report, with which filters, and when.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `reports.sales` | Manager, Admin |
| `reports.inventory` | Manager, Admin |
| `reports.stitching` | Manager, Admin, Counter Staff (limited) |
| `reports.finance` | Admin, Accountant |
| `reports.tax` | Admin, Accountant |
| `dashboard.view` | All authenticated (scoped to role) |

---

## Dependencies

- All modules: reports query their data.
- Module 36 (P&L): `pl_snapshots` table for dashboard.
- Module 08 (Audit Log): export events logged.

---

## Out of Scope (MVP)

- Custom report builder.
- Scheduled report email delivery.
- Charts and graphs (dashboard uses text/number widgets; charts are a Phase 3 feature).
- BI tool integration (Metabase, Power BI).
- Comparison to prior period (this month vs last month) — data is available; the comparison UI is deferred.
