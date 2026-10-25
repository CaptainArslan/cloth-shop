# Module 29 — Unified Calendar

**Status:** Draft  
**Phase:** Planning  
**Last updated:** 2026-10-25

---

## Purpose & Scope

A single calendar view that shows all time-sensitive events: stitching order delivery promises, ironing pickup dates, vendor job expected returns, and alteration promised dates. Staff use it to see what is due, what is overdue, and how loaded any given day is before accepting new orders.

---

## Entities & Key Fields

No dedicated tables. The calendar aggregates data from:
- `stitching_orders.promised_delivery_date`
- `stitching_order_lines.promised_date`
- `ironing_orders.promised_pickup_date`
- `vendor_jobs.expected_return_date`
- `alteration_orders.promised_date`
- `country_holidays` (Module 02)

### `calendar_events` (virtual / query-built)

The calendar is built from queries, not stored events. A summary record may be cached for performance (Redis or a nightly-built summary table) but is not the source of truth.

---

## Features

### Day view

Shows all promises for a selected date:
- Stitching orders due for delivery
- Ironing pickups
- Vendor returns expected
- Alteration pickups

Items are colour-coded by status (overdue = red, on-track = green, due today = orange).

### Week / month view

Shows the count of items due per day. Hovering a day shows the breakdown by type.

### Hijri calendar overlay

If `calendar.show_hijri = true` (from settings), each date shows both Gregorian and Hijri. Important for Eid-based planning (Module 30).

### Vendor workload view

A sub-view showing, per vendor, how many pieces are due back each day. Used to decide whether to issue more work to a vendor.

---

## Business Rules

1. **Read-only display.** The calendar displays existing data; dates are changed through the originating order, not the calendar.
2. **Holiday markers.** Public holidays and weekends (from Module 02) are highlighted. Staff are warned if they set a promised date on a holiday (the booking flow calls a `CalendarService::isWorkingDay()` check).
3. **Filter by order type, status, and staff member.** The calendar can be filtered so each role sees what is relevant to them.
4. **Overdue highlighting.** Items past their promised date that are not yet delivered/collected are highlighted in red.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `calendar.view` | All authenticated |
| `calendar.view_vendor` | Manager, Admin |

---

## Dependencies

- Module 23, 26, 27, 25: sources of dated events.
- Module 02 (Country Packs): holidays and weekend days.
- Module 09 (Shop Settings): `calendar.show_hijri`.
- Module 30 (Capacity & Booking Control): capacity limits used in the calendar day view.

---

## Out of Scope (MVP)

- Drag-and-drop rescheduling from the calendar.
- Export to Google Calendar / iCal.
- Staff shift scheduling on the calendar.
