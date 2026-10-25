# Module 10 — Customers

**Status:** Draft  
**Phase:** Parties  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages the customer master record and family profile structure. In a cloth shop, one phone number often represents an entire family — father, mother, children — each with their own measurements and order history. The module stores contact details, family members, credit limits, balance, and the full interaction history.

Scope: customer profiles, family members, phone-based lookup, credit management, customer search, and history view. Measurements live in Module 11.

---

## Entities & Key Fields

### `customers`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `branch_id` | FK | Branch where the customer was created; they can transact at any branch |
| `name` | string(255) | Primary contact name (head of family / account holder) |
| `phone` | string | E.164 format; unique per tenant. Primary lookup key at counter |
| `phone_verified` | boolean | True once an OTP/SMS is confirmed (ecommerce) |
| `alternate_phone` | string | nullable |
| `email` | string | nullable |
| `preferred_language` | char(5) | BCP-47; for receipts and notifications |
| `address` | JSON | nullable; {street, area, city, country} |
| `id_type` | enum | nullable; `national_id` \| `passport` \| `resident_card` |
| `id_number` | string | nullable; encrypted at rest |
| `tax_exempt` | boolean | Default false; if true, tax engine skips all taxes for this customer |
| `credit_limit` | bigint | Minor units in tenant base currency; 0 = no credit |
| `credit_balance` | bigint | Current outstanding balance (positive = customer owes us) |
| `notes` | text | Internal notes; not shown to customer |
| `active` | boolean | |
| `created_by` | FK → users | |
| `created_at` / `updated_at` | timestamps | |

### `customer_family_members`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `customer_id` | FK | The account holder (parent record) |
| `name` | string(255) | e.g. "Ahmed (son)", "Fatima (wife)" |
| `relation` | string | nullable; free text or enum: self, spouse, son, daughter, other |
| `gender` | enum | `male` \| `female` \| `unspecified` |
| `approximate_age` | tinyint | nullable; useful for sizing |
| `notes` | text | nullable |
| `active` | boolean | |
| `created_at` / `updated_at` | timestamps | |

> The account holder themselves can optionally be a family member (`relation = self`) to hold their own measurements. This is the default when a new customer is created.

### `customer_tags` (pivot)

Flexible tagging for segmentation and reporting.

| Column | Type | Notes |
|--------|------|-------|
| `customer_id` | FK | |
| `tag` | string | e.g. "VIP", "Bulk buyer", "Seasonal" |

---

## Business Rules

1. **Phone as primary key.** Phone number is the lookup key at the POS counter. It must be unique per tenant. Entering a phone that already exists opens the existing customer record rather than creating a duplicate.
2. **Family member required.** When placing a stitching order, a family member must be selected (even if it's "self"). This links measurements and order history to the correct person.
3. **Credit limit enforcement.** If `pos.credit_limit_enforced = true` in settings, a sale that would push `credit_balance` above `credit_limit` is blocked. Manager override (with PIN) bypasses this for the current transaction.
4. **Credit balance tracking.** `credit_balance` is updated by the ledger module (Module 31 — Customer Ledger). The customer record itself holds a denormalized balance for fast POS lookup; the source of truth is the ledger.
5. **ID number encryption.** `id_number` is encrypted at rest (Laravel `encrypted` cast). Never logged in plaintext; audit log stores `[REDACTED]` for this field.
6. **Active flag.** Deactivating a customer prevents new sales and orders but preserves all history.
7. **Merge duplicates.** If two customer records are found to be the same person (entered twice with slightly different names), an admin can merge them: all orders, measurements, and ledger entries are re-pointed to the surviving record; the duplicate is marked inactive with a `merged_into` FK.

---

## Workflows

### New customer at POS counter

1. Cashier types phone number.
2. System checks: if found, open existing record. If not found, prompt to create.
3. Creation form: name, phone, optional email, preferred language.
4. A `customer_family_members` record with `relation = self` is created automatically.
5. Customer saved; POS proceeds to sale.

### Adding a family member

1. At any point (during an order or from the customer profile), staff selects "Add family member".
2. Enters name, relation, gender, approximate age.
3. New `customer_family_members` record saved.
4. Measurements can now be taken for this member (Module 11).

### Customer lookup / search

Search by: phone (exact or partial), name (partial), ID number (exact). Results are scoped to the current tenant.

### Merging duplicates

1. Admin finds the two records.
2. Selects "Merge" — picks the surviving record.
3. System re-assigns all linked records (orders, measurements, ledger entries, family members) to the survivor.
4. Duplicate record: `active = false`, `merged_into = survivor_id`.
5. Audit log records the merge with both IDs.

---

## Customer History View

A single customer profile page aggregates:

- All stitching orders (across all family members), with status.
- All POS sales.
- All payments and outstanding balance.
- All measurements (per family member).
- All WhatsApp/SMS notifications sent.
- Audit trail of profile changes.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `customers.view` | Cashier, Manager, Admin |
| `customers.create` | Cashier, Manager, Admin |
| `customers.update` | Cashier (own-session created), Manager, Admin |
| `customers.deactivate` | Manager, Admin |
| `customers.merge` | Admin |
| `customers.view_id_number` | Manager, Admin (`audit.view_sensitive`) |
| `customers.set_credit_limit` | Admin, Accountant |
| `customers.override_credit` | Manager (with PIN), Admin |

---

## Dependencies

- Module 01 (Tenant & Branch): `tenant_id`, `branch_id`.
- Module 03 (Multilingual): `preferred_language` for receipts and notifications.
- Module 04 (Currency): `credit_limit` and `credit_balance` in minor units.
- Module 05 (Tax Engine): `tax_exempt` flag consumed by the engine.
- Module 08 (Audit Log): all changes, especially ID number access.
- Module 11 (Measurements): each family member can have measurement records.
- Module 31 (Customer Ledger): source of truth for balance; customer record caches it.
- Module 37 (Notifications): preferred language used when sending messages.

---

## Edge Cases

- **Same phone, different tenant.** Phone uniqueness is per-tenant. The same phone number can exist in two tenants.
- **Phone number change.** Rare but possible (customer changes SIM). Requires manager permission. Old phone kept in an `alternate_phone` field and audit log.
- **Family member with separate phone.** Family members do not have their own phone number — they are always under the account holder's phone. If a son wants a separate account later, a new `customers` record is created for him.
- **No-name customer.** Some POS sales are anonymous (walk-in, cash). A system "Walk-in" customer exists per branch (seeded on branch creation) for these sales. It has no credit, no family members, no measurements. Stitching orders cannot be placed against it.
- **Customer with zero credit limit.** `credit_limit = 0` means no credit allowed — any unpaid balance must be collected at time of sale.

---

## Out of Scope (MVP)

- Customer self-service portal / account login.
- Loyalty points / rewards programme.
- Customer-facing order tracking link.
- GDPR / data deletion request workflow (framework is in place via active flag and audit; formal deletion process is a later phase).
- Customer import from CSV / Excel.
