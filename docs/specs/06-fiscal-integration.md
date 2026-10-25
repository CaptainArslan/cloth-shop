# Module 06 — Fiscal Integration (FBR / Oman e-invoicing / None)

**Status:** Draft  
**Phase:** Foundation  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Provides a pluggable contract for fiscal/tax-authority integration. The application never calls a fiscal provider directly — it calls a `FiscalProviderInterface` and the container resolves the correct implementation based on the tenant's `fiscal_provider` setting. This keeps all fiscal logic isolated and testable, and makes adding a new country's integration a matter of writing one new class.

Scope: provider interface, null provider (default/Oman), FBR POS provider (Pakistan), Oman e-invoice provider (future), fiscal status tracking on invoices, retry queue, and fiscal block on receipts.

---

## Provider Interface

```php
interface FiscalProviderInterface
{
    /**
     * Submit an invoice to the fiscal authority.
     * Returns a FiscalResult with status, fiscal number, QR payload, raw response.
     */
    public function submitInvoice(Invoice $invoice): FiscalResult;

    /**
     * Check submission status for an invoice already submitted.
     * Used by the retry queue.
     */
    public function checkStatus(Invoice $invoice): FiscalResult;

    /**
     * Whether this provider requires a fiscal block before printing a receipt.
     * If true, receipt printing is blocked until submitInvoice() returns success.
     */
    public function requiresFiscalBlockBeforePrint(): bool;

    /**
     * Returns the provider's identifier string.
     */
    public function identifier(): string;
}
```

### `FiscalResult`

```php
class FiscalResult
{
    public FiscalStatus $status;      // pending | success | failed | not_required
    public ?string      $fiscalNumber;
    public ?string      $qrPayload;   // base64-encoded QR data per authority spec
    public ?array       $rawRequest;
    public ?array       $rawResponse;
    public ?string      $errorCode;
    public ?string      $errorMessage;
}
```

---

## Implementations

### `NullFiscalProvider` (default — Oman MVP)

- `submitInvoice()` immediately returns `FiscalResult(status: not_required)`.
- `requiresFiscalBlockBeforePrint()` returns `false`.
- No network calls; no storage of fiscal data.
- Used when `fiscal_provider = 'null'` or not configured.

### `FbrPosProvider` (Pakistan)

- Submits sale invoices to FBR's POS integration API in real time.
- `requiresFiscalBlockBeforePrint()` returns `true` — receipt cannot be printed until FBR confirms and returns a fiscal number.
- Stores FBR invoice number, QR code payload, and raw JSON request/response on the `invoices` record.
- On failure, places the invoice in the retry queue (see below) and blocks the receipt.
- FBR requires: seller NTN/STRN, branch registration code, invoice date/time, buyer CNIC (optional), line items with HSN codes, tax amounts.

### `OmanEInvoiceProvider` (future — Fawtara)

- Oman's Fawtara e-invoicing system (verify current rollout scope — may not apply to retail).
- Not in MVP. Registered in the container as a stub that returns `not_required` until implemented.

---

## Entities & Key Fields

### `invoices` (fiscal columns)

Added to every billable document's table (sale, stitching order, etc.):

| Column | Type | Notes |
|--------|------|-------|
| `fiscal_provider` | string | Snapshot of provider at time of creation |
| `fiscal_status` | enum | `not_required` \| `pending` \| `submitted` \| `confirmed` \| `failed` |
| `fiscal_number` | string | nullable; assigned by authority |
| `fiscal_qr_payload` | text | nullable; base64 QR data |
| `fiscal_submitted_at` | timestamp | nullable |
| `fiscal_confirmed_at` | timestamp | nullable |
| `fiscal_raw_request` | JSON | nullable |
| `fiscal_raw_response` | JSON | nullable |
| `fiscal_error_code` | string | nullable |
| `fiscal_error_message` | text | nullable |
| `fiscal_attempts` | tinyint | Retry count |

### `fiscal_retry_queue`

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `tenant_id` | FK | |
| `document_type` | string | Polymorphic |
| `document_id` | ULID | |
| `provider` | string | |
| `attempt_number` | tinyint | |
| `next_attempt_at` | timestamp | Exponential backoff |
| `last_error` | text | nullable |
| `created_at` | timestamp | |

---

## Business Rules

1. **Provider per tenant.** The provider is resolved from `tenants.fiscal_provider`. It is bound in the container at boot time via a factory that reads the current tenant.
2. **Fiscal block.** If `requiresFiscalBlockBeforePrint()` is true and fiscal status is not `confirmed`, the POS blocks receipt printing and shows a message. The cashier cannot hand over the receipt. (Exception: offline mode — see Module 41.)
3. **Async submission.** In Pakistan, FBR submission happens synchronously at the moment of sale (FBR requirement for real-time POS). If the FBR API is unreachable, the job is queued for retry.
4. **Retry policy.** Retry up to 5 times with exponential backoff (1 min, 5 min, 15 min, 1 hr, 4 hr). After 5 failures, the invoice is marked `failed` and an alert is sent to the tenant admin. The invoice can be manually resubmitted.
5. **Raw request/response stored.** For audit and debugging. Sensitive fields (e.g. customer CNIC) may need to be redacted per the authority's data retention rules.
6. **Fiscal number on receipt.** When `fiscal_number` is present, it is printed on the receipt along with the QR code (if provided). For `NullFiscalProvider`, neither is printed.
7. **Offline mode.** If the POS is offline (Module 41), fiscal submission is queued for when connectivity is restored. FBR mandates submission within a grace period (verify the current FBR offline tolerance window).
8. **Provider change.** Changing `fiscal_provider` on a tenant takes effect for new invoices only. Historical invoices retain their `fiscal_provider` snapshot.

---

## Workflows

### Sale with FBR fiscal block

1. Cashier finalises the sale.
2. System calls `FbrPosProvider::submitInvoice($invoice)`.
3. FBR API responds within ~2 seconds with a fiscal number and QR payload.
4. System stores response, sets `fiscal_status = confirmed`.
5. Receipt is printed with fiscal number and QR.

### Sale with FBR — API unreachable

1. System calls `FbrPosProvider::submitInvoice($invoice)`.
2. HTTP timeout or error response received.
3. `fiscal_status = pending`, invoice added to `fiscal_retry_queue`.
4. If `requiresFiscalBlockBeforePrint()` is true: receipt is held. POS shows "Fiscal submission pending — receipt will print automatically when confirmed." Alternatively (depending on FBR rules), a provisional receipt with "PENDING FBR SUBMISSION" watermark may be printed.
5. Retry job runs in the background; on success, receipt becomes printable.

### Manual resubmission

Tenant admin navigates to the failed invoice, reviews the error, and clicks "Resubmit". System places it back in the queue with `attempt_number = 1`.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `fiscal.view_status` | Cashier, Manager, Admin |
| `fiscal.resubmit` | Manager, Admin |
| `fiscal.configure_provider` | Tenant admin |

---

## Dependencies

- Module 01 (Tenant & Branch): `fiscal_provider` on tenant; `fiscal_branch_code` on branch.
- Module 02 (Country Packs): `fiscal_provider` seeded from pack.
- Module 05 (Tax Engine): tax amounts and line details passed to the provider.
- Module 08 (Audit Log): all fiscal submissions and failures audited.
- Module 21 (POS): fiscal block enforced at checkout.
- Module 41 (Offline POS & Sync): offline queuing of fiscal submissions.

---

## Edge Cases

- **Provider mis-match.** If the pack sets `fiscal_provider = fbr` but no FBR credentials are configured in settings, provider instantiation fails with a clear exception at boot. Do not silently fall back to Null.
- **Duplicate submission.** If a retry job runs and FBR has already accepted the invoice (network timeout on response), FBR may return a "duplicate" error. Provider must handle this by checking for the existing fiscal number and marking the invoice `confirmed` rather than `failed`.
- **Refund/return invoices.** FBR requires credit notes to reference the original fiscal number. `FbrPosProvider::submitInvoice()` must handle `document_type = credit_note` differently.
- **Oman Fawtara uncertainty.** Until the applicability of Fawtara to retail cloth shops is confirmed, `OmanEInvoiceProvider` is a stub. If it becomes mandatory, it must be implemented before going live in Oman.

---

## Out of Scope (MVP)

- Oman Fawtara / e-invoice implementation (stub only).
- VAT return filing.
- B2B e-invoicing (buyer TRN on invoice — UI field exists, but e-invoice API not called in MVP).
- Multiple fiscal providers on one tenant (e.g. per-branch FBR registration with different STRNs — the `fiscal_branch_code` column on `branches` supports this, but the provider logic assumes one STRN per tenant for MVP).
