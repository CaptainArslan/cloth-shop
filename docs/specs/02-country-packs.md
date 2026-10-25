# Module 02 — Country Packs & Localization

**Status:** Draft  
**Phase:** Foundation  
**Last updated:** 2026-10-25

---

## Purpose & Scope

A Country Pack is a seedable configuration bundle that sets every country-specific default for a tenant in one operation. It prevents hard-coding Oman or Pakistan assumptions into application code and makes it straightforward to onboard a new country.

This module covers: currency defaults, tax defaults, phone format, weekend days, holidays, payment methods, garment templates, courier adapters, and fiscal provider selection. The localisation of UI strings is handled in Module 03 (Multilingual).

---

## Entities & Key Fields

### `country_packs` (config files, not a DB table)

Country packs live in `config/country_packs/{country_code}.php` (or YAML). They are loaded once at tenant-creation time and written into `tenant_settings` / `branch_settings`. After that, the application reads settings — it never reads the pack file at runtime.

Each pack file exports:

```php
return [
    'country_code'        => 'OM',
    'currency_code'       => 'OMR',
    'currency_subunit'    => 1000,       // 1 OMR = 1000 baisa (3 decimal places)
    'base_language'       => 'ar',
    'rtl'                 => true,
    'timezone'            => 'Asia/Muscat',
    'phone_format'        => '+968 XXXX XXXX',
    'phone_country_code'  => '+968',
    'weekend_days'        => ['friday', 'saturday'],  // ISO day names
    'date_format'         => 'DD/MM/YYYY',
    'fiscal_provider'     => 'null',    // 'null' | 'fbr' | 'oman_e_invoice'
    'tax_rates'           => [...],     // see Module 05
    'payment_methods'     => [...],     // see Module 21
    'garment_templates'   => [...],     // see Module 11
    'courier_adapters'    => [...],     // see Module 40
    'public_holidays'     => [...],     // ISO dates or recurring rules
];
```

### `country_holidays`

| Column | Type | Notes |
|--------|------|-------|
| `id` | bigint | |
| `country_code` | char(2) | |
| `name` | string | Translatable key |
| `date` | date | For fixed-date holidays |
| `hijri_month` / `hijri_day` | tinyint | For Hijri-based holidays (Eid); nullable |
| `is_recurring` | boolean | If true, applies every year on this month/day |
| `year` | year | nullable; for one-off holidays |
| `tenant_id` | FK | nullable; null = applies to all tenants in that country |

### `payment_methods`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `code` | string | cash, card, bank_transfer, qr_pay, knet, sadad, easypaisa, jazzcash |
| `name` | JSON | Translatable: {en, ar, ur} |
| `is_cash_equivalent` | boolean | Affects day-end cash closing |
| `active` | boolean | |
| `sort_order` | tinyint | |

---

## Built-in Country Packs

### Oman (OM)

| Setting | Value |
|---------|-------|
| Currency | OMR, 3 decimal places |
| Base language | Arabic (RTL) |
| Timezone | Asia/Muscat (UTC+4) |
| Weekend | Friday–Saturday |
| Tax | 5% VAT (inclusive or exclusive, configurable per product) |
| Fiscal | `NullFiscalProvider` (Fawtara TBD — see open question #6) |
| Payment methods | Cash, card (KNET/Visa/MC), bank transfer, QR Pay (Oman) |
| Garment templates | Dishdasha, Kumma, Abaya, Bisht, Agal, Mussar |
| Couriers | Aramex, Oman Post |
| Calendar | Hijri + Gregorian dual display |

### Pakistan (PK)

| Setting | Value |
|---------|-------|
| Currency | PKR, 2 decimal places (subunit = 100; 1 rupee = 100 paisa; stored as paisa, displayed as rupees) |
| Base language | Urdu (RTL) |
| Timezone | Asia/Karachi (UTC+5) |
| Weekend | Sunday (Saturday half-day in some businesses) |
| Tax | FED/GST on goods (federal) + provincial tax on services |
| Fiscal | `FbrPosProvider` |
| Payment methods | Cash, card, EasyPaisa, JazzCash, bank transfer |
| Garment templates | Shalwar Kameez, Kurta, Pant-shirt, Waistcoat, Dupatta |
| Couriers | PostEx, Leopards, TCS, Rider |
| Calendar | Gregorian; Hijri for Eid holidays only |

---

## Business Rules

1. **Pack applied once.** A country pack is applied at tenant creation and seeds defaults. Subsequent changes to tenant settings are made directly in `tenant_settings`, not by re-applying the pack.
2. **Pack is a floor, not a ceiling.** Every default seeded from a pack can be overridden per tenant or per branch.
3. **Holiday resolution order.** When checking if a date is a holiday: tenant-specific holidays first, then country-level holidays. Tenant can add holidays and suppress country defaults.
4. **Hijri calendar.** For Eid holidays, the system stores the Hijri month and day. On display, it converts using a Hijri library (e.g. `iamkun/dayjs` with hijri plugin, or a PHP Hijri converter). The actual Gregorian date is confirmed and stored when it is officially announced.
5. **Weekend days affect.** Capacity planning (Module 30), calendar availability (Module 29), and Eid booking cutoffs. Weekend days from the pack seed the calendar default; tenant can override.
6. **Currency subunit.** The `currency_subunit` integer (1000 for OMR, 100 for PKR) tells every money calculation how many minor units equal one major unit. All amounts in the database are stored in minor units (baisa for OMR, paisas for PKR). See Module 04.

---

## Workflow: Onboarding a New Country

1. Developer creates `config/country_packs/{code}.php` with all required keys.
2. Developer adds garment templates for that country to the templates seeder.
3. Developer registers the country's fiscal provider (or maps it to `NullFiscalProvider`).
4. Developer adds the country's holiday rules to the holidays seeder.
5. QA: creates a test tenant with that country code and verifies that all settings, tax rates, payment methods, and templates are seeded correctly.
6. No application code changes should be required for a new country.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `country_packs.apply` | Super-admin (at tenant creation) |
| `holidays.manage` | Tenant admin |
| `payment_methods.manage` | Tenant admin |

---

## Dependencies

- Module 01 (Tenant & Branch): pack is applied to a tenant on creation.
- Module 04 (Currency): subunit and currency code from the pack feed the money engine.
- Module 05 (Tax Engine): tax rates from the pack seed the initial rate table.
- Module 06 (Fiscal Integration): `fiscal_provider` from the pack selects the provider binding.
- Module 11 (Measurements): garment templates from the pack seed the template library.
- Module 21 (POS): payment methods from the pack seed the POS payment options.
- Module 29 (Calendar): weekend days and holidays from the pack seed the calendar defaults.

---

## Edge Cases

- **Eid date not yet announced.** Store the Hijri date; display as "Eid (date TBC)" until the Gregorian date is confirmed and stored. Booking cutoffs cannot be calculated until confirmed.
- **Country not in built-in packs.** System falls back to a `generic` pack: OMR as currency, no tax, Gregorian calendar, no fiscal provider. Admin must configure everything manually.
- **Pack version drift.** If the pack file is updated after tenants are seeded, existing tenants are not automatically updated. A migration script or admin action is needed to push specific setting changes.
- **Dual weekend.** Some Oman businesses work Saturday mornings. The pack stores full/half-day weekend rules; capacity planning respects them.

---

## Out of Scope (MVP)

- UI for browsing or editing country pack definitions (packs are code/config files).
- Automatic Eid date lookup from an external API.
- Currency subunit changes (e.g. PKR redenomination) — treat as a new currency code.
- Multi-country tenant (one tenant operating in two countries simultaneously).
