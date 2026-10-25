# Module 07 — Users, Roles & Permissions

**Status:** Draft  
**Phase:** Foundation  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages authentication, authorisation, and access control for all system users. Role-based access control (RBAC) with per-branch scope. A user may have different roles in different branches.

Scope: user accounts, roles, permissions, branch assignment, PIN-based POS login (fast counter switching), session management, and super-admin separation.

---

## User Types

| Type | Description |
|------|-------------|
| **Super-admin** | Platform-level; can manage tenants. Lives outside tenant scope. |
| **Tenant admin** | Full access to their tenant across all branches. |
| **Branch manager** | Full access to their assigned branch(es). |
| **Cashier** | POS operations for their branch. |
| **Tailor / Karigar** | Read job slips; update piece status. |
| **Driver / Delivery** | View and update delivery assignments. |
| **Accountant** | Read/write financial records; no POS. |
| **Viewer** | Read-only access to reports. |

These are default roles seeded from the country pack. Tenant admin can create custom roles.

---

## Entities & Key Fields

### `users`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | nullable for super-admin |
| `name` | string | |
| `email` | string | Unique per tenant (not globally — same email may exist in two tenants) |
| `phone` | string | E.164; nullable |
| `password` | string | Bcrypt |
| `pos_pin` | string | 4–6 digit PIN, hashed; nullable. Used for fast POS login |
| `language` | char(5) | BCP-47; user's preferred UI language |
| `active` | boolean | Deactivated users cannot log in |
| `last_login_at` | timestamp | nullable |
| `created_at` / `updated_at` | timestamps | |

### `roles`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | nullable for system roles |
| `name` | string | Unique per tenant |
| `is_system` | boolean | System roles cannot be deleted |
| `description` | text | nullable |

### `permissions`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `name` | string | Unique globally; dot-namespaced: `orders.create`, `cash.close_day` |
| `group` | string | UI grouping: `Orders`, `Finance`, `Inventory`, … |
| `description` | text | |

### `role_permissions` (pivot)

`role_id` + `permission_id`

### `user_branch_roles`

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `user_id` | FK | |
| `branch_id` | FK | nullable for tenant-wide roles |
| `role_id` | FK | |

A user can have:
- A tenant-wide role (`branch_id = null`) — applies to all branches.
- Branch-specific roles — applies only when they are operating that branch.
- Both — branch-specific role takes precedence (additive: union of permissions).

---

## Permission Naming Convention

`{resource}.{action}` — all lowercase, dot-separated.

Examples: `customers.view`, `customers.create`, `orders.cancel`, `cash.close_day`, `reports.financial`, `discounts.above_10_percent`, `settings.manage`.

Permissions are defined in code (`PermissionSeeder`) and are immutable in production. Roles are flexible and can be customised by the tenant admin.

---

## POS PIN Login

The POS screen has two login modes:

1. **Full login**: email + password. Used once per shift to establish a session.
2. **PIN login**: 4–6 digit PIN. Used to switch between cashiers without logging out the session. Does not create a new HTTP session; sets a `pos_active_user_id` in the existing session.

PIN is hashed (bcrypt). Failed PIN attempts are rate-limited (5 attempts per 2 minutes per terminal).

---

## Business Rules

1. **Tenant isolation.** A user's `tenant_id` is immutable. Users cannot see or access another tenant.
2. **Permission check.** All gates check the user's resolved permissions for the active branch. `Gate::allows('orders.create')` resolves permissions from the user's roles for the current branch.
3. **Super-admin bypass.** Super-admins bypass all tenant gates. They are identified by `tenant_id = null` and a `is_super_admin` flag.
4. **Deactivation.** Deactivating a user invalidates their sessions and blocks all future logins. Their historical records (created_by, etc.) are preserved.
5. **Role deletion.** System roles cannot be deleted. Custom roles can be deleted only if no active user has that role.
6. **Minimum one admin.** A tenant must always have at least one active user with the Tenant Admin role. Removing the last admin is blocked.
7. **Discount limits by role.** Roles carry a `max_discount_percent` setting (nullable = no limit). The POS enforces this when a cashier applies a discount. Overrides require a manager PIN.
8. **Audit.** All permission and role changes are written to the audit log (Module 08).

---

## Workflows

### Creating a user (tenant admin)

1. Admin enters name, email, phone, role(s), branch assignment(s), language.
2. System sends a welcome email with a password-set link (expires 24 h).
3. User sets password; account becomes active.

### Assigning a role

1. Admin selects user → Roles tab.
2. Selects branch (or "All branches") and role.
3. System validates: no duplicate `(user, branch, role)`.
4. Change is written to `user_branch_roles` and the audit log.

### Manager override at POS

1. Cashier attempts an action above their permission (e.g. discount > 10%).
2. POS prompts "Manager PIN required".
3. Manager enters their PIN.
4. Action proceeds and is logged with the authorising user's ID.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `users.view` | Manager, Admin |
| `users.create` | Admin |
| `users.update` | Admin |
| `users.deactivate` | Admin |
| `roles.view` | Manager, Admin |
| `roles.manage` | Admin |
| `permissions.view` | Admin |

---

## Dependencies

- Module 01 (Tenant & Branch): users scoped to tenant; roles scoped to branch.
- Module 08 (Audit Log): user and role changes audited.
- All other modules: every action checks a permission from this module.

---

## Edge Cases

- **User in two branches with conflicting roles.** Permissions are additive (union) across all of the user's roles for the current branch context.
- **Super-admin impersonation.** Super-admin can impersonate a tenant user for support. All actions taken under impersonation are logged as `(super-admin-id impersonating user-id)`.
- **Email change.** Email is the login identifier. Changing it requires re-verification. Old email is kept for 48 h as an alias to handle in-flight emails.
- **PIN forgotten.** Admin resets the PIN from the user management screen. PIN reset is logged.
- **Concurrent sessions.** A user may be logged in from multiple devices. Deactivation invalidates all sessions immediately (delete all sessions from `sessions` table keyed by `user_id`).

---

## Out of Scope (MVP)

- SSO / OAuth (Google, Microsoft).
- Two-factor authentication (deferred to security hardening phase).
- Fine-grained record-level permissions (e.g. "can only see their own customers").
- API tokens / personal access tokens (for the ecommerce API).
- LDAP / Active Directory integration.
