# Module 37 — Notifications (WhatsApp / SMS)

**Status:** Draft  
**Phase:** Communication  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Sends automated and manual messages to customers via WhatsApp (primary) or SMS (fallback) at key order lifecycle events. Messages are templated, multi-language, and logged. An adapter pattern allows switching or adding providers without changing business logic.

---

## Notification Events

| Event | Trigger | Default on? |
|-------|---------|------------|
| Stitching order booked | Order created | Yes |
| Order ready for pickup | All lines → ready | Yes |
| Ironing ready | Ironing → done | Yes |
| Alteration ready | Alteration → done | Yes |
| Balance reminder | Balance due > 0 and order delivered | Optional |
| Uncollected order (1st) | N days after promised date | Yes |
| Uncollected order (2nd/3rd) | Repeat interval | Yes |
| Payment received | Payment recorded | Optional |
| Order cancelled | Order cancelled | Optional |

---

## Entities & Key Fields

### `notification_templates`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `event` | string | e.g. `order.booked`, `order.ready`, `uncollected.reminder_1` |
| `channel` | enum | `whatsapp` \| `sms` |
| `language` | char(5) | BCP-47 |
| `subject` | string | nullable; for channels that support it |
| `body` | text | Template with `{variables}` placeholders |
| `is_active` | boolean | |

> One template per (event, channel, language). The system picks the template matching the customer's `preferred_language`; falls back to `en`.

### `notification_log`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` | FK | |
| `customer_id` | FK | |
| `event` | string | |
| `channel` | string | |
| `phone` | string | Destination number |
| `language` | char(5) | |
| `body_sent` | text | Rendered message |
| `status` | enum | `queued` \| `sent` \| `delivered` \| `failed` |
| `provider_response` | JSON | nullable |
| `sent_at` | timestamp | nullable |
| `created_at` | timestamp | |

---

## Provider Adapter

```php
interface NotificationProviderInterface
{
    public function sendWhatsApp(string $phone, string $body): NotificationResult;
    public function sendSms(string $phone, string $body): NotificationResult;
}
```

Implementations:
- `TwilioProvider` — WhatsApp via Twilio API (or WhatsApp Business API directly).
- `UltraMsgProvider` — WhatsApp via UltraMsg.
- `NullNotificationProvider` — for development/testing; logs without sending.

The active provider is set in Shop Settings and resolved from the container.

---

## Template Variables

Common variables available in all templates:

| Variable | Example |
|----------|---------|
| `{customer_name}` | Ahmed Al-Balushi |
| `{order_number}` | ST-00042 |
| `{garment_count}` | 3 |
| `{promised_date}` | 15 Ramadan 1446 / 14 March 2025 |
| `{balance_due}` | OMR 12.500 |
| `{shop_name}` | Al-Noor Fashion |
| `{shop_phone}` | +968 1234 5678 |
| `{branch_name}` | Main Branch |

Date variables are formatted in the customer's locale (Gregorian + Hijri if `calendar.show_hijri = true`).

---

## Business Rules

1. **Customer opt-out.** If a customer's `receive_notifications = false`, no messages are sent to them. (Field added to `customers` table.)
2. **Phone validation.** Messages are only sent to verified E.164 phone numbers. Invalid or missing phones are skipped and logged as failed.
3. **Queue-based.** All notification sends are queued (Laravel Queue). Failures are retried up to 3 times. After 3 failures, status = `failed` and an admin alert is generated.
4. **Manual send.** Staff can manually trigger a notification from the order detail screen (e.g. "Send ready notification again"). This creates a `notification_log` entry with `event = manual`.
5. **Rate limiting.** No more than 3 automated messages per customer per day (configurable). This prevents spamming if there are multiple uncollected orders.
6. **WhatsApp Business template compliance.** WhatsApp Business API requires pre-approved message templates for outbound messages outside the 24-hour window. Templates are approved once; the system uses them. If a message falls within a 24-hour active window, free-form messages are allowed.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `notifications.send_manual` | Cashier, Manager, Admin |
| `notification_templates.manage` | Admin |
| `notification_log.view` | Manager, Admin |

---

## Dependencies

- Module 10 (Customers): `preferred_language`, `phone`, `receive_notifications`.
- Module 03 (Multilingual): template language selection.
- Module 09 (Shop Settings): `notifications.channel`, per-event on/off flags.
- Module 23, 27, 26, 28: trigger events for automated notifications.

---

## Out of Scope (MVP)

- Email notifications.
- Push notifications (mobile app).
- WhatsApp template approval workflow (done manually via the WhatsApp Business Manager).
- Bulk marketing campaigns.
- Two-way WhatsApp conversations (customer replies handled manually).
