# Module 18 — Stock Transfers

**Status:** Draft  
**Phase:** Inventory  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Handles the physical movement of fabric rolls, remnants, and other stock between branches of the same tenant. Ensures that inventory records at both branches remain accurate and that the transfer is audited.

Scope: transfer requests, dispatch, receive, and roll status updates. Cash transfers between accounts are in Module 35.

---

## Entities & Key Fields

### `stock_transfers`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `from_branch_id` | FK | |
| `to_branch_id` | FK | |
| `transfer_number` | string | Auto-generated; unique per tenant |
| `status` | enum | `draft` \| `dispatched` \| `received` \| `cancelled` |
| `requested_by` | FK → users | |
| `dispatched_by` | FK → users | nullable |
| `received_by` | FK → users | nullable |
| `dispatched_at` | timestamp | nullable |
| `received_at` | timestamp | nullable |
| `notes` | text | nullable |
| `created_at` / `updated_at` | timestamps | |

### `stock_transfer_lines`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `stock_transfer_id` | FK | |
| `fabric_roll_id` | FK | The specific roll being transferred |
| `transferred_length` | decimal(8,3) | Metres being transferred; ≤ current remaining_length |
| `received_length` | decimal(8,3) | nullable; actual length confirmed on receipt (may differ) |
| `notes` | text | nullable |

---

## Business Rules

1. **Whole-roll or partial transfer.** A roll can be transferred entirely or partially. If partial, a `roll_cuts` record (`cut_type = stock_audit`) is created at the source branch for the transferred portion; a new `fabric_rolls` record is created at the destination for that portion.
2. **Roll is locked during transit.** When a transfer is `dispatched`, the roll at the source branch is flagged `status = transferred` (for a full transfer) or its `remaining_length` is decremented (for a partial transfer). It cannot be cut or sold while in transit.
3. **Receipt confirmation required.** The transfer is not complete until the destination branch confirms receipt. This prevents the stock from disappearing or being double-counted.
4. **Length discrepancy on receipt.** If the received length differs from the dispatched length, the discrepancy is noted in `received_length` vs `transferred_length`. A discrepancy report flags this for investigation (counting error or in-transit loss).
5. **Cost carries over.** The new roll record at the destination branch inherits the `cost_per_unit` from the source roll. The transfer itself does not generate a financial entry (same-entity transfer); no P&L impact.
6. **Cross-branch only.** Transfers within the same branch (e.g. between shelves) are not recorded in this module — they are just location updates on the roll record.

---

## Workflow

1. Branch manager at the destination requests a transfer (or the source branch initiates one).
2. Source branch staff reviews the request, selects rolls and quantities, marks as `dispatched`.
3. Source rolls are flagged or decremented.
4. Physical goods move.
5. Destination branch receives and confirms quantities.
6. New roll records created at destination; source rolls updated (exhausted, or remaining length reduced).

---

## Permissions

| Permission | Who can |
|------------|---------|
| `stock_transfers.create` | Manager, Admin |
| `stock_transfers.dispatch` | Manager, Admin (source branch) |
| `stock_transfers.receive` | Manager, Admin (destination branch) |

---

## Dependencies

- Module 01 (Tenant & Branch): transfers are intra-tenant, cross-branch.
- Module 16 (Roll Inventory): roll status updated on dispatch and receipt.
- Module 08 (Audit Log): all transfer events audited.

---

## Out of Scope (MVP)

- Inter-tenant transfers (consignment between separate shops).
- Transfer of non-roll stock (piece items) — data model supports it but UI is fabric-roll-focused in MVP.
- Automated transfer suggestions (low stock at one branch, surplus at another).
