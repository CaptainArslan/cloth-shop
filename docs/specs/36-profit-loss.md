# Module 36 — Profit & Loss

**Status:** Draft  
**Phase:** Finance  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Provides the income statement (P&L) for a selected period and branch. Calculates gross profit from sales and stitching income, deducts COGS (cost of fabric sold) and vendor costs, deducts operating expenses, and shows net profit. All figures in the tenant base currency; multi-currency amounts converted at the rate locked on each transaction.

---

## P&L Structure

```
Revenue
  ├── Fabric sales (from POS)
  ├── Stitching charges (from stitching orders)
  ├── Ironing income (from ironing orders)
  ├── Alteration charges
  └── Other income

Cost of Goods Sold (COGS)
  └── Cost of fabric sold (from roll cost_per_unit × metres sold)

Gross Profit = Revenue - COGS

Operating Expenses
  ├── Vendor payments (stitching costs paid to darzi)
  ├── Salaries & wages (from employee salary records)
  ├── Rent
  ├── Utilities
  ├── Marketing
  ├── Petty cash & miscellaneous
  └── Other expenses (from expense categories)

Operating Profit = Gross Profit - Operating Expenses

Other Income / Expense
  ├── Exchange gain / loss
  └── One-time items

Net Profit = Operating Profit ± Other
```

---

## Entities & Key Fields

No dedicated tables. The P&L is computed on demand from:
- `sale_lines` (revenue + COGS)
- `stitching_order_lines` (stitching revenue)
- `ironing_order_lines` (ironing revenue)
- `vendor_ledger_entries` (vendor cost)
- `expenses` (operating expenses, grouped by category)
- `account_transactions` (exchange gain/loss entries)

A summary `pl_snapshots` table is generated nightly for quick report loading:

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `tenant_id` / `branch_id` | FK | |
| `period_year` | smallint | |
| `period_month` | tinyint | |
| `revenue_fabric` | bigint | |
| `revenue_stitching` | bigint | |
| `revenue_ironing` | bigint | |
| `cogs` | bigint | |
| `gross_profit` | bigint | |
| `vendor_costs` | bigint | |
| `salary_costs` | bigint | |
| `other_expenses` | bigint | |
| `net_profit` | bigint | |
| `currency_code` | char(3) | Tenant base currency |
| `generated_at` | timestamp | |

---

## Business Rules

1. **COGS per line.** When a sale line is saved, the `cost_per_unit` from the `fabric_rolls` record at the time of the cut is snapshotted onto the sale line as `cost_amount`. This is the exact fabric cost for that cut.
2. **Multi-currency conversion.** All revenue and cost figures are converted to the tenant base currency using the rate locked on each transaction. No current-day rate is used for historical figures.
3. **Branch and tenant level.** P&L can be shown per branch or rolled up across all branches of the tenant.
4. **Period selection.** Any date range is supported. Pre-built options: today, this week, this month, this quarter, this year, custom range.
5. **Snapshot for dashboard.** The nightly snapshot feeds the dashboard (Module 38) so the home screen loads instantly. Real-time P&L (computed on demand) is available from the full report screen.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `pl_report.view` | Manager (own branch), Admin, Accountant |
| `pl_report.view_all_branches` | Admin, Accountant |

---

## Dependencies

- All revenue modules: Module 21, 23, 26, 27.
- Module 16 (Roll Inventory): COGS via `cost_per_unit`.
- Module 32 (Vendor Ledger): vendor cost.
- Module 34 (Expenses): operating expenses.
- Module 35 (Cash Management): exchange gain/loss.
- Module 04 (Currency): all conversions to base currency.

---

## Out of Scope (MVP)

- Full balance sheet (assets, liabilities, equity) — data is partially available; formal balance sheet requires a chart of accounts, deferred.
- Cash flow statement.
- Budget vs actual comparison.
- Department/cost-centre level P&L.
