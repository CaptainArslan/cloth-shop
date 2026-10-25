# Module 40 — Online Orders & Couriers

**Status:** Draft  
**Phase:** Online  
**Last updated:** 2026-10-25

---

## Purpose & Scope

Manages the fulfilment and shipment of online orders placed via the ecommerce storefront (Module 39). Integrates with courier APIs to create shipments, generate waybills, track parcels, and handle COD (cash on delivery) reconciliation.

---

## Courier Adapters

Same adapter pattern as fiscal and notification providers:

```php
interface CourierProviderInterface
{
    public function createShipment(OnlineOrder $order): ShipmentResult;
    public function trackShipment(string $trackingNumber): TrackingResult;
    public function cancelShipment(string $trackingNumber): bool;
}
```

| Provider | Market | Notes |
|----------|--------|-------|
| `AramexProvider` | Oman | International + local |
| `OmanPostProvider` | Oman | Local only |
| `PostExProvider` | Pakistan | COD-heavy |
| `LeopardsProvider` | Pakistan | COD-heavy |
| `NullCourierProvider` | Dev/test | Logs without sending |

Active provider set in Shop Settings per branch.

---

## Entities & Key Fields

### `shipments`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `online_order_id` | FK | |
| `courier_provider` | string | Snapshot of active provider |
| `tracking_number` | string | Assigned by courier on shipment creation |
| `waybill_url` | string | nullable; URL to printable waybill |
| `status` | enum | `created` \| `picked_up` \| `in_transit` \| `out_for_delivery` \| `delivered` \| `returned` \| `cancelled` |
| `estimated_delivery_date` | date | nullable |
| `actual_delivery_date` | date | nullable |
| `cod_amount` | bigint | nullable; COD collection amount (minor units) |
| `cod_collected` | boolean | |
| `cod_remitted` | boolean | Whether courier has paid COD back to shop |
| `last_tracking_update` | JSON | Raw tracking response |
| `created_at` / `updated_at` | timestamps | |

---

## Business Rules

1. **Shipment created after order is packed.** Staff marks the order as "ready to ship" after packing. The system calls the courier API and creates a shipment. The tracking number is stored and the waybill is printed.
2. **Tracking updates.** A scheduled job polls the courier API every few hours for status updates on active shipments. Status is updated on the `shipments` record and a notification is sent to the customer if the status changes to `out_for_delivery` or `delivered`.
3. **COD reconciliation.** For COD orders, `cod_amount` is the amount the courier must collect from the customer. When the courier remits the cash to the shop, `cod_remitted = true` and a `account_transactions` entry is created in Module 35.
4. **Failed delivery.** If the courier marks a shipment as `returned`, the order status is updated and the customer is notified. Staff decide whether to reship or refund.
5. **Customer tracking link.** The tracking number is included in the "shipped" notification to the customer. For public tracking (courier's own tracking page), the link is constructed from the tracking number.

---

## Workflow

1. Online order confirmed and paid.
2. Staff picks and packs the order.
3. Staff marks order as "ready to ship" in the admin.
4. System creates shipment via courier API. Waybill printed (Module 20).
5. Courier picks up parcel.
6. System tracks status automatically.
7. Customer receives "out for delivery" notification.
8. Delivered: `cod_collected` set; COD remittance tracked.

---

## Permissions

| Permission | Who can |
|------------|---------|
| `shipments.create` | Manager, Admin |
| `shipments.view` | Cashier, Manager, Admin |
| `shipments.cod_reconcile` | Admin, Accountant |

---

## Dependencies

- Module 39 (Ecommerce): orders to fulfil.
- Module 20 (Labels): waybill printing.
- Module 35 (Cash): COD remittance entry.
- Module 37 (Notifications): shipping status notifications.

---

## Out of Scope (MVP)

- Multi-parcel shipments (one order = multiple boxes).
- Returns portal for customers to request a return and get a courier label.
- Courier rate comparison at checkout.
- Automated courier selection based on destination / weight.
