# Module 41 — Offline POS & Sync

**Status:** Draft  
**Phase:** Technical  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Enables the POS to continue operating when the internet connection is lost. Transactions are queued locally and synced to the server when connectivity is restored. Fiscal submissions that require real-time confirmation (FBR) are handled with grace-period receipts during offline periods.

---

## Architecture

The POS is a Vue.js SPA. Offline capability is achieved via:

1. **Service Worker** — caches the POS app shell and static assets so the UI loads without a network.
2. **IndexedDB (local storage)** — stores: the product catalog (last synced), roll inventory (last known remaining lengths), customer lookup cache, and queued transactions.
3. **Background Sync API** — when connectivity returns, the browser's Background Sync API triggers the sync queue to flush to the server.

> This is a progressive enhancement. The POS works with full features online. Offline mode degrades gracefully (see degraded features below).

---

## What Works Offline

| Feature | Offline support |
|---------|----------------|
| Fabric sales (from cached products) | Yes |
| Customer lookup (cached customers) | Yes (cached only; new customers cannot be created offline) |
| Payment recording | Yes; queued |
| Receipt printing | Yes (printed locally; fiscal number may be pending) |
| Stitching order creation | Yes; queued |
| Roll length decrement | Optimistic (decremented locally; reconciled on sync) |
| Fiscal submission (FBR) | Queued; provisional receipt printed |
| Inventory lookup (new rolls added since last sync) | No |

---

## Sync Protocol

### Outbound (local → server)

Queued operations (transactions, payments, order creates) are replayed against the server API in chronological order when connectivity returns. The server API is idempotent: each operation carries a client-generated UUID that prevents duplicate processing if a sync is retried.

### Inbound (server → local)

After each outbound sync, the POS pulls:
- Updated product prices (effective today).
- Updated roll remaining lengths (reflecting cuts made on other terminals).
- New customers added elsewhere (up to last-sync window).

### Conflict resolution

**Roll length conflict**: if the local cache shows 5.0m remaining on a roll, but the server says 3.0m (another terminal cut 2.0m while offline), the local queue's cut is replayed against 3.0m. If the local cut was for 4.0m (which would make remaining negative), the sync fails for that transaction. The cashier is shown an error: "Roll now has insufficient length. Please re-process this sale." The sale is marked `sync_failed` in the queue for manual resolution.

### Sync status indicator

The POS shows a connection status indicator: green (online), amber (syncing), red (offline, N transactions queued).

---

## Fiscal Offline Handling

**NullFiscalProvider (Oman):** No impact — fiscal submissions are not required.

**FbrPosProvider (Pakistan):** FBR requires real-time submission; offline mode is a compliance edge case. During offline:
1. Sale is recorded locally.
2. A provisional receipt is printed with "PENDING FBR SUBMISSION" watermark.
3. The fiscal submission is queued for when connectivity returns.
4. On sync, the FBR submission is made; if successful, the fiscal number is stored and a second receipt can be printed (or the status is accessible on the customer's order).
5. FBR defines an allowed offline grace period (verify the current FBR offline tolerance policy); the system flags any submission that falls outside this window for manual review.

---

## Entities & Key Fields

### `sync_queue` (IndexedDB, client-side only)

| Field | Type | Notes |
|-------|------|-------|
| `client_id` | UUID | Unique per operation |
| `operation_type` | string | `create_sale`, `create_order`, `record_payment`, etc. |
| `payload` | JSON | Full operation data |
| `created_at` | timestamp | |
| `sync_status` | string | `queued` \| `synced` \| `failed` |
| `error` | string | nullable |

### `sync_failures` (server-side)

Operations that failed reconciliation on the server are stored here for admin review:

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `branch_id` | FK | |
| `client_id` | UUID | |
| `operation_type` | string | |
| `payload` | JSON | |
| `failure_reason` | text | |
| `resolved` | boolean | |
| `created_at` | timestamp | |

---

## Business Rules

1. **Idempotent API.** Every write endpoint accepts a `client_id` (UUID). If the same `client_id` is received twice, the server returns the result of the first operation without re-processing.
2. **Cash session.** Cash sessions are opened and closed offline using local state. On sync, the server reconciles. A mismatch (e.g. the server shows the session was already closed by a manager) creates a sync failure for admin review.
3. **No new roll creation offline.** Receiving goods (Module 17) requires connectivity. This is a planned operational constraint — advise the shop to ensure connectivity before receiving a delivery.
4. **Time drift.** The POS clock may drift from the server clock during offline periods. Transaction timestamps are stored as client-local time and a `client_timezone` is sent to the server for accurate recording.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `sync_failures.view` | Manager, Admin |
| `sync_failures.resolve` | Admin |

---

## Dependencies

- Module 21 (POS): offline sales queued.
- Module 23 (Stitching Orders): offline orders queued.
- Module 06 (Fiscal): fiscal submissions queued.
- Module 35 (Cash): cash sessions reconciled on sync.
- Module 16 (Roll Inventory): conflict resolution for roll lengths.

---

## Out of Scope (MVP)

- Offline stitching job management (Module 25 requires connectivity).
- Offline new customer creation.
- Multi-terminal conflict resolution beyond roll lengths (e.g. two terminals create overlapping stitching orders for the same vendor on the same day — handled manually post-sync).
- Native mobile app (PWA with service worker covers the offline use case for tablet-based POS).
