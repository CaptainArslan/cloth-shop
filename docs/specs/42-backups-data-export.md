# Module 42 — Backups & Data Export

**Status:** Draft  
**Phase:** Technical  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Ensures the shop's data is backed up regularly, restorable quickly, and exportable by the tenant for their own records or migration to another system. Covers automated database backups, file storage backups, backup verification, and full data export in open formats.

---

## Backup Strategy

### Database backups

| Frequency | Retention | Storage |
|-----------|-----------|---------|
| Daily (full dump) | 30 days | Remote object storage (S3-compatible: Cloudflare R2 or Wasabi for cost) |
| Weekly (full dump) | 3 months | Same storage, different prefix |
| Monthly (full dump) | 1 year | Cold storage tier |

Backups are triggered by a scheduled Laravel job. The dump is compressed (gzip), encrypted at rest (AES-256 with a key stored separately from the backup files), and uploaded to the remote bucket.

### File/storage backups

Uploaded files (product images, design photos, generated PDFs) are stored in a Laravel disk. The disk is backed up with the same schedule as the database.

### Backup verification

A weekly job downloads the most recent daily backup, decompresses it, and performs a test restore to a temporary database. If the restore succeeds, the backup is marked `verified`. If it fails, an alert is sent to the system admin.

---

## Entities & Key Fields

### `backup_logs`

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `tenant_id` | FK | nullable for system-level backups |
| `backup_type` | enum | `database` \| `files` |
| `frequency` | enum | `daily` \| `weekly` \| `monthly` |
| `status` | enum | `running` \| `success` \| `failed` \| `verified` |
| `file_path` | string | Remote path to the backup file |
| `file_size_bytes` | bigint | |
| `checksum` | string | SHA-256 of the backup file |
| `started_at` / `completed_at` | timestamps | |
| `error_message` | text | nullable |

---

## Data Export

Tenant admins can request a full export of their data in open formats. The export includes:

| Data | Format |
|------|--------|
| Customers + family members | CSV |
| Measurements | CSV |
| Sales | CSV |
| Stitching orders | CSV |
| Vendor ledger | CSV |
| Supplier ledger | CSV |
| Customer ledger | CSV |
| Expenses | CSV |
| Products & price history | CSV |
| Roll inventory history | CSV |
| Audit log | CSV |

The export is a ZIP archive containing all CSV files. The admin requests it from the admin panel; the system generates it asynchronously (queued job) and sends a download link via email when ready.

---

## Business Rules

1. **Tenant-scoped export.** A tenant's export contains only their data — not other tenants'. The export job runs with a tenant scope to prevent leakage.
2. **Backup encryption.** Encryption key is stored in Laravel Vapor secrets or `.env` on the server — not in the database. The backup is unusable without the key.
3. **Backup alerts.** If a backup fails, an alert is sent to the system admin email immediately. If three consecutive backups fail, an escalation alert is sent.
4. **Retention and deletion.** Backups older than their retention period are deleted from remote storage automatically by a lifecycle rule on the object storage bucket (not by application code — a safety measure against accidental deletion via the app).
5. **Point-in-time restore.** With daily backups, the maximum data loss in a disaster is one day. Point-in-time recovery (via database WAL archiving) is a later enhancement.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `backups.view` | Super-admin |
| `backups.trigger_manual` | Super-admin |
| `data_export.request` | Tenant admin |
| `data_export.download` | Tenant admin (within 24h of generation) |

---

## Dependencies

- All modules: data export covers all tenant-scoped data.
- Module 08 (Audit Log): export requests logged.

---

## Out of Scope (MVP)

- Point-in-time recovery (WAL archiving / continuous backup).
- Automated restore testing for each tenant's data separately (system-level verification only).
- Selective export (export only a date range or module subset).
- Data migration import (importing from another system).
- Geo-redundant backup (secondary region) — single region in MVP.
