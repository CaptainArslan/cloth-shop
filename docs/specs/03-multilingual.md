# Module 03 — Multilingual (Arabic / English / Urdu, RTL)

**Status:** Draft  
**Phase:** Foundation  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Provides the language and text-direction infrastructure for all user-facing content: UI strings, database-stored translatable fields, receipt/invoice templates, notification templates, and garment template names.

Scope: locale resolution, RTL layout support, translatable model fields, translation file management, and receipt/notification rendering in the customer's preferred language.

---

## Supported Locales

| Code | Language | Direction | Primary market |
|------|----------|-----------|----------------|
| `ar` | Arabic | RTL | Oman (default) |
| `en` | English | LTR | Both markets |
| `ur` | Urdu | RTL | Pakistan |

The tenant sets a `default_language`. Users can set their own UI language. Customers have a preferred language used for receipts, notifications, and the ecommerce storefront.

---

## Translatable Database Fields

Fields that need multilingual content (names, descriptions, template labels) are stored as JSON columns:

```json
{ "en": "Dishdasha", "ar": "دشداشة", "ur": "ڈشڈاشہ" }
```

### Laravel implementation

Use a translatable trait (e.g. `Spatie\Translatable` or a custom `HasTranslations` trait) that:
- Serialises JSON on write.
- Deserialises and returns the correct locale string on read, falling back to `en` if the requested locale is missing.
- Exposes `getTranslations('name')` to retrieve the full JSON.
- Supports `->setTranslation('name', 'ar', 'دشداشة')`.

All translatable columns are suffixed `_translations` at the database level (to distinguish from non-translatable string columns) or use a `{field}` JSON column with documented convention — follow existing project convention once it is established.

### Which fields are translatable

| Entity | Translatable fields |
|--------|-------------------|
| `tax_categories` | `name` |
| `tax_rates` | `name` |
| `payment_methods` | `name` |
| `product_catalog` items | `name`, `description` |
| `garment_templates` | `name`, measurement label names |
| `notification_templates` | `subject`, `body` |
| `branches` | `name` (for customer-facing display) |
| `expense_categories` | `name` |

---

## UI Language Resolution

1. User's saved `language` preference (from their profile) — highest priority.
2. Browser `Accept-Language` header (for unauthenticated pages, ecommerce).
3. Tenant `default_language` — fallback.
4. `en` — ultimate fallback.

Laravel's locale is set per-request via middleware:

```php
// SetLocaleMiddleware
app()->setLocale($resolvedLocale);
Carbon::setLocale($resolvedLocale);
```

---

## RTL Layout

Both Arabic and Urdu are RTL. The frontend (Vue.js) and receipt/PDF renderer must support RTL.

### Frontend (Vue.js)

- `dir="rtl"` on `<html>` when locale is `ar` or `ur`.
- Tailwind CSS: use logical properties (`ms-`, `me-`, `ps-`, `pe-`, `start-`, `end-`) instead of `ml-`, `mr-`, `pl-`, `pr-`, `left-`, `right-`. This ensures layouts flip automatically.
- Icon mirroring: arrow icons must be mirrored in RTL (use `rtl:scale-x-[-1]`).
- Number formatting: Arabic locale uses Eastern Arabic numerals (٠١٢٣٤٥٦٧٨٩) in some contexts. Default to Western Arabic (0–9) throughout for POS displays; use Eastern Arabic only in formatted Arabic text if explicitly requested.

### Receipts & PDFs

- Use a PDF library that supports RTL text (e.g. `barryvdh/laravel-dompdf` with RTL support, or `mPDF`).
- Test each receipt template in all three locales.
- Number alignment: numbers are always LTR even inside an RTL document.

---

## Translation File Management

### Backend (Laravel)

Translation strings live in `lang/{locale}/` PHP files or JSON files per locale:
- `lang/ar/messages.php`
- `lang/en/messages.php`
- `lang/ur/messages.php`

Use namespaced keys: `messages.orders.status.ready` not bare strings.

### Frontend (Vue.js)

Use `vue-i18n`. Translation files in `resources/lang/{locale}.json`. Keys mirror the backend naming convention.

### Adding a new string

1. Add the English key and value in `lang/en/`.
2. Add corresponding keys in `lang/ar/` and `lang/ur/`. If translation is pending, use the English string as a placeholder prefixed with `[TODO]` — this is a visible flag to translators.
3. For database-stored translatable fields, always provide at least the `en` key; `ar` and `ur` are populated by the tenant admin or a translator.

---

## Customer-Facing Language

- Customers have a `preferred_language` field on their profile (Module 10).
- Receipts, notifications (WhatsApp/SMS), and ecommerce pages use the customer's language.
- If the customer has no preference, the branch's default language is used.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `translations.manage` (in-app string overrides) | Tenant admin |
| UI language selection | Each user (own profile) |
| Customer language | Set at customer creation or customer self-service |

---

## Dependencies

- Module 01 (Tenant & Branch): `default_language` on tenant.
- Module 02 (Country Packs): base language from country pack seeds tenant default.
- Module 10 (Customers): `preferred_language` on customer.
- Module 37 (Notifications): notification templates per locale.

---

## Edge Cases

- **Missing translation key.** Laravel fallback chain: requested locale → `en` → key itself (visible bug indicator in development; in production, return `en` silently).
- **Partial translatable field.** If only `en` is filled for a name, the Arabic UI returns the English name. The admin panel should highlight untranslated fields.
- **Mixed RTL/LTR in same document.** A Urdu receipt with an English product name: use `dir="auto"` on inline spans so the browser/renderer handles bidi correctly.
- **Urdu numerals.** By default, Urdu output uses Western Arabic numerals (0–9) for prices on receipts, matching cash register expectations. If the tenant wants Eastern Arabic/Indic numerals, it's a setting.

---

## Out of Scope (MVP)

- Machine translation integration.
- In-app translation editor UI for string overrides (tenant can edit via admin settings; a full translation editor is a later phase).
- Right-to-left PDF generation for Urdu (Arabic PDF is higher priority; Urdu PDF fallbacks to LTR if the PDF library doesn't support Urdu well).
- Language variants (e.g. Egyptian Arabic vs Gulf Arabic).
