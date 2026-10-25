# Module 08 — Audit Log

**Status:** Draft  
**Phase:** Foundation  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Immutable, append-only record of every significant state change in the system. Provides accountability, regulatory traceability, and an investigation trail for disputes or errors. Every module writes to the audit log; this module defines the schema, retention policy, and viewing interface.

Scope: audit log schema, what must be logged, writer API, viewer/search interface, retention, and tamper evidence.

---

## What Must Be Logged

### Always logged (mandatory)

| Category | Events |
|----------|--------|
| Auth | Login, logout, failed login, PIN login, impersonation start/end, password change |
| Users & Roles | User created/updated/deactivated, role assigned/removed, permission changed |
| Financial | Invoice created, payment recorded, payment voided, cash session opened/closed, exchange rate changed, discount applied (with authorising user if overridden), refund issued |
| Tax | Tax rate created/updated/deactivated |
| Fiscal | Submission attempt, confirmation, failure, manual resubmission |
| Inventory | Stock adjustment (with reason), stock transfer, roll cut, wastage recorded |
| Orders | Order created, status changed, order cancelled, delivery confirmed, uncollected order reminder sent |
| Settings | Any setting changed (key, old value, new value) |
| Branch/Tenant | Branch created, deactivated; tenant setting changed |

### Logged on configurable sensitivity level

| Category | Default | Notes |
|----------|---------|-------|
| POS view events | Off | Can be enabled for high-security environments |
| Report exports | On | Who ran what report and when |
| Customer profile views | Off | Privacy consideration; off by default |

---

## Entities & Key Fields

### `audit_logs`

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint UNSIGNED AUTO_INCREMENT | Monotonic insert order |
| `tenant_id` | FK | nullable for super-admin events |
| `branch_id` | FK | nullable for tenant-level events |
| `user_id` | FK → users | nullable for system/background events |
| `user_name_snapshot` | string | Name at time of action (in case user is later deleted) |
| `impersonated_by` | FK → users | nullable; set when super-admin impersonates |
| `event` | string | Dot-namespaced: `orders.status_changed`, `cash.session_closed` |
| `auditable_type` | string | Morph; model class name |
| `auditable_id` | ULID / bigint | ID of the affected record |
| `old_values` | JSON | nullable; previous state (only changed fields) |
| `new_values` | JSON | nullable; new state (only changed fields) |
| `metadata` | JSON | nullable; contextual data (IP, device, branch code, POS terminal ID) |
| `created_at` | timestamp | Immutable; the time of the action |

### Indexes

- `(tenant_id, created_at)` — primary listing index.
- `(auditable_type, auditable_id)` — look up history of a specific record.
- `(user_id, created_at)` — user activity report.
- `(event, created_at)` — event-type filtering.

---

## Writer API

All application code writes to the audit log through a service, never directly:

```php
AuditLog::record(
    event:          'orders.status_changed',
    auditable:      $order,
    oldValues:      ['status' => 'pending'],
    newValues:      ['status' => 'ready'],
    metadata:       ['branch_code' => 'MB'],
);
```

The service resolves `tenant_id`, `branch_id`, `user_id`, and `metadata.ip` from the current request context automatically. Background jobs pass the triggering user's ID explicitly.

### Laravel Observer / Model Events

For Eloquent models marked with `Auditable` (a trait), the observer automatically records `created`, `updated`, and `deleted` events. Only changed fields are stored in `old_values` / `new_values`. Sensitive columns (passwords, PINs, tokens) are excluded by a `$auditExclude` array on the model.

For business events that don't map to a single Eloquent event (e.g. a payment that spans multiple tables), `AuditLog::record()` is called explicitly in the service layer.

---

## Business Rules

1. **Append-only.** No audit log record is ever updated or deleted by application code. Database-level: the application DB user has no `UPDATE` or `DELETE` privilege on `audit_logs`.
2. **No personal data in values.** `old_values` and `new_values` must not store plaintext passwords, PINs, API keys, or customer payment card data. The `$auditExclude` list on each model enforces this.
3. **Retention.** Default: 7 years (matching typical VAT record-keeping requirements). Configurable per tenant in settings. After the retention period, records may be archived to cold storage (not deleted, unless explicitly required by a data deletion request under applicable privacy law).
4. **Tamper evidence (MVP-level).** Each record gets a checksum of its own fields stored in a `checksum` column. A background job can verify integrity. Full cryptographic chaining (each record's hash includes the previous record's hash) is deferred to a later phase.
5. **Performance.** Writes to `audit_logs` must not add latency to the request. Use `defer()` or queue the write for all non-critical audit events. Financial and fiscal events are written synchronously (within the same DB transaction) to ensure they are never lost.
6. **Sensitive event masking.** The viewer masks sensitive fields (`customer.phone`, `customer.id_number`) from users who don't have the `audit.view_sensitive` permission.

---

## Viewer Interface

### Admin panel audit log screen

- Filterable by: date range, event type, user, branch, auditable type/ID.
- Shows: timestamp, user, event label (human-readable), summary of changes.
- Clicking a row expands to show old vs new values side by side.
- Exportable to CSV for external audit use.

### Record history (contextual)

Any model with the `Auditable` trait shows a "History" tab in its detail view listing only that record's audit entries.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `audit.view` | Manager, Admin |
| `audit.view_sensitive` | Admin only |
| `audit.export` | Admin |

---

## Dependencies

- Module 07 (Users, Roles & Permissions): `user_id` and impersonation context.
- Module 01 (Tenant & Branch): `tenant_id`, `branch_id`.
- All other modules: write to the audit log.

---

## Edge Cases

- **Background job with no user.** `user_id = null`; `event` includes the job class name in `metadata.job`. Example: `notifications.reminder_sent` by the `SendUndeliveredOrderReminderJob`.
- **Cascade deletes.** If a related record is deleted and the audit log references it via `auditable_id`, the log record is not deleted. The `user_name_snapshot` and similar snapshot fields preserve context even after the source record is gone.
- **High-frequency write models.** Some models (e.g. real-time stock level after every cut) would generate too many audit records. Use debouncing: log the first change and the final state at the end of the session/day, not every intermediate change.
- **Audit log insert failure.** For synchronous writes (financial events), if the audit insert fails, rollback the parent transaction. For deferred writes, log the failure to the application error log and alert operations — do not silently lose the event.

---

## Out of Scope (MVP)

- Cryptographic hash chaining (blockchain-style tamper-proofing).
- Automated compliance report generation (GDPR, VAT audit trail reports).
- Real-time alerting on suspicious audit events (anomaly detection).
- External SIEM integration.
- Automated archival to cold storage (manual export only in MVP).
