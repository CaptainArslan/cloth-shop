# Module 01 — Tenant & Branch Management

**Status:** Draft  
**Phase:** Foundation (required before all other modules)  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Provides the top-level organisational structure that every other module hangs from. A **tenant** represents a single business (one cloth shop, or one SaaS customer if the product is later sold). A **branch** is a physical location within that tenant.

Every piece of stock, every sale, every employee, and every financial record belongs to a branch. Reporting can be at branch level or rolled up to tenant level.

This module also holds the decision point for multi-tenancy: the schema must support it from day one even if the first deployment is single-tenant.

---

## Entities & Key Fields

### `tenants`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | Primary key |
| `name` | string(255) | Business/shop name |
| `slug` | string(100) | URL-safe identifier; unique globally |
| `country_code` | char(2) | ISO 3166-1 alpha-2 (OM, PK, …) |
| `default_currency_code` | char(3) | ISO 4217 (OMR, PKR) |
| `default_language` | char(5) | BCP-47 (ar, en, ur) |
| `timezone` | string | IANA tz (Asia/Muscat, Asia/Karachi) |
| `fiscal_provider` | string | Enum: null \| fbr \| oman_e_invoice |
| `plan` | string | Subscription plan (for SaaS billing later) |
| `active` | boolean | Soft-disable without deleting |
| `created_at` / `updated_at` | timestamps | |

### `branches`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | Primary key |
| `tenant_id` | FK → tenants | |
| `name` | string(255) | e.g. "Main Branch", "Ruwi Outlet" |
| `code` | string(20) | Short code used in invoice series, labels (e.g. MB, RW) |
| `country_code` | char(2) | May differ from tenant (future cross-border) |
| `currency_code` | char(3) | Operating currency for this branch |
| `address` | JSON | Structured: street, city, region, postal_code, country |
| `phone` | string | E.164 format |
| `email` | string(255) | nullable |
| `fiscal_branch_code` | string | nullable; FBR branch registration / Oman branch ID |
| `is_default` | boolean | Exactly one default per tenant |
| `active` | boolean | |
| `created_at` / `updated_at` | timestamps | |

### `tenant_settings` (key-value store per tenant)

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `tenant_id` | FK | |
| `key` | string | Namespaced, e.g. `pos.receipt_footer` |
| `value` | text | JSON-encoded |

> Branch-level overrides for settings live in `branch_settings` with the same shape plus `branch_id`.

---

## Business Rules

1. **Tenant isolation.** All queries in application code must scope to the resolved tenant. A global scope (`TenantScope`) is applied on every model that carries `tenant_id`. Cross-tenant queries are only allowed in super-admin context.
2. **Branch required.** No sale, stock record, or financial entry may exist without a `branch_id`. Nullable `branch_id` is a bug.
3. **Default branch.** Each tenant must have exactly one default branch (`is_default = true`). Enforced via a unique partial index and a service method that swaps the flag atomically.
4. **Active flag.** Deactivating a tenant hides it from login but retains all data. Deactivating a branch prevents new transactions but historical data remains.
5. **Slug uniqueness.** `tenants.slug` is globally unique and immutable after creation. Used for subdomain routing in SaaS mode (`{slug}.clothshop.com`).
6. **Country code drives defaults.** When a branch is created, it inherits the tenant's `country_code`, `currency_code`, and `timezone` unless overridden.

---

## Multi-Tenancy Strategy

**Decision required** (open question #1): own-shop only, or SaaS?

Regardless of that decision, the database must be built for multi-tenancy from the start. Chosen approach:

- **Single database, `tenant_id` column on every tenant-scoped table.** This is the lowest-cost first step and can be migrated to per-schema or per-database later if needed.
- A `TenantResolver` middleware resolves the current tenant from subdomain (SaaS) or a fixed config value (single-tenant) and binds it into the container.
- Eloquent global scope `TenantScope` automatically adds `WHERE tenant_id = ?` to all queries.
- Super-admin routes bypass the scope via `withoutTenantScope()`.

---

## Workflows

### Creating a tenant (super-admin)

1. Super-admin submits: name, slug, country, currency, language, timezone, fiscal provider.
2. System creates `tenants` record and a first `branches` record (`is_default = true`, inheriting country/currency).
3. System creates the first admin user for that tenant.
4. System seeds country-specific defaults (tax rates, garment templates, payment methods) via the Country Pack for that tenant.

### Creating a branch

1. Tenant admin submits branch details.
2. If `is_default = true`, system clears `is_default` on the existing default branch in the same transaction.
3. Branch becomes available for assignment to users.

### Deactivating a branch

1. System checks for open stitching orders, pending deliveries, or unclosed cash sessions on that branch.
2. If any exist, deactivation is blocked with a list of blockers.
3. Once clear, `active = false` is set; no new transactions are allowed.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `tenants.create` | Super-admin only |
| `tenants.update` | Super-admin |
| `tenants.deactivate` | Super-admin |
| `branches.create` | Tenant admin |
| `branches.update` | Tenant admin |
| `branches.deactivate` | Tenant admin |
| `branches.view` | All authenticated users in the tenant |

---

## Dependencies

- **None.** This is a root module; all others depend on it.
- Country Packs (Module 02) seeds defaults after tenant creation.
- Users, Roles & Permissions (Module 07) creates the first admin user.

---

## Edge Cases

- **Slug collision on creation.** Return a validation error; suggest an alternative (append a number).
- **Only branch being deactivated.** Block: a tenant must always have at least one active branch.
- **Currency mismatch.** If a branch currency differs from tenant currency, all financial roll-ups must convert using the exchange rate at time of transaction (see Module 04).
- **Tenant deleted.** Hard delete is not allowed; use deactivate. Data retention for regulatory purposes (tax records, employment records) must survive deactivation.
- **Single-tenant mode.** When `MULTI_TENANT=false` in `.env`, the resolver returns a fixed tenant and the slug/subdomain routing is skipped.

---

## Out of Scope (MVP)

- Per-schema or per-database tenant isolation.
- Tenant self-registration / onboarding wizard.
- SaaS billing and plan enforcement.
- Cross-branch stock transfers (covered in Module 18).
- Tenant-level branding (logo, colours) — deferred to later phase.
