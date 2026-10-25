# Module 39 — Ecommerce Storefront

**Status:** Draft  
**Phase:** Online  
**Last updated:** 2026-10-25

---

## Purpose & Scope

A customer-facing web storefront for browsing and buying fabric, viewing the design gallery, and booking stitching appointments. Stock is synced in real time from the main inventory. Orders placed online feed directly into the POS/stitching order system.

Scope: product listing, fabric browsing, cart, checkout, online stitching appointment booking, and order status tracking. Courier integration is in Module 40.

---

## Storefront Pages

| Page | Content |
|------|---------|
| Home | Featured fabrics, banners, new arrivals |
| Shop / Fabric listing | Browse by category, brand, price, colour |
| Product detail | Fabric details, price per metre, roll image, available length, buy |
| Design gallery | Filtered by garment type, occasion (from Module 24) |
| Stitching booking | Select garment type, design, measurements (if logged in), schedule appointment |
| Cart | Review items, quantities |
| Checkout | Address, payment, order confirmation |
| My orders | Order history, status, tracking |
| My measurements | View and update measurements (if logged in) |

---

## Entities & Key Fields

### `online_customers`

Ecommerce customers authenticate separately from POS staff. An online customer is linked to a `customers` record via their phone/email verification.

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `customer_id` | FK → customers | Linked after phone verification |
| `email` | string | Login credential |
| `password` | string | Hashed |
| `phone_verified` | boolean | |
| `created_at` | timestamp | |

### `online_orders`

| Column | Type | Notes |
|--------|------|-------|
| `id` | ULID | |
| `tenant_id` / `branch_id` | FK | Fulfilling branch |
| `online_customer_id` | FK | |
| `customer_id` | FK | Linked `customers` record |
| `type` | enum | `fabric_purchase` \| `stitching_appointment` |
| `status` | enum | `pending_payment` \| `confirmed` \| `processing` \| `shipped` \| `delivered` \| `cancelled` |
| `total_amount` | bigint | |
| `payment_status` | enum | `unpaid` \| `paid` \| `refunded` |
| `payment_gateway` | string | e.g. `thawani` (Oman), `stripe`, `cod` |
| `payment_reference` | string | Gateway transaction ID |
| `delivery_address` | JSON | |
| `notes` | text | nullable |
| `created_at` | timestamp | |

---

## Stock Sync

The storefront shows real-time fabric availability from the `fabric_rolls` table. Key rules:

1. **Available length** shown = sum of `remaining_length` across all `in_stock` rolls for that product at the fulfilling branch.
2. **Cart reservation.** When a customer adds fabric to cart, no stock is reserved (too complex for MVP). The cart shows "available" at the time of browsing. Stock is confirmed and decremented only when the order is confirmed.
3. **Out-of-stock handling.** If a customer tries to check out a quantity that is no longer available (another customer bought it first), they see an error and can adjust their quantity.

---

## Online Stitching Appointment Booking

1. Customer selects garment type and design (optional).
2. System shows available appointment slots (from Module 30 calendar capacity).
3. Customer picks a slot and provides measurements (if not previously saved) or uploads their own.
4. Order is created as `stitching_appointment` type. A notification is sent to the shop.
5. Staff reviews and confirms the appointment (creates a `stitching_orders` record internally).
6. Customer receives confirmation with appointment date and advance payment link.

---

## Payment Gateway

Oman: Thawani Pay (local Omani payment gateway supporting card, STC Pay, Apple Pay).  
Pakistan: JazzCash, EasyPaisa, or Stripe (to be confirmed).  
COD (Cash on Delivery): available for local delivery orders.

A `PaymentGatewayInterface` follows the same adapter pattern as fiscal and notification providers.

---

## Business Rules

1. **Online order → internal order.** Every confirmed online fabric purchase creates a `sale` record (Module 21) internally. Every confirmed stitching appointment creates a `stitching_orders` record (Module 23). The online order ID is stored as a reference on the internal record.
2. **Customer account link.** When an online customer places their first order, the system checks if a `customers` record exists for their phone number. If yes, the account is linked. If no, a new `customers` record is created.
3. **Arabic/RTL.** The storefront fully supports Arabic and RTL layout. Language is selected by the customer; the browser default is used for first visit.
4. **Product visibility.** Only `active` products with available stock are shown. Products with `is_public = false` are hidden. Design gallery entries with `is_public = false` are hidden.
5. **Pricing.** Prices shown on the storefront are the branch's active price for the product, tax-inclusive display (for Oman: VAT included).

---

## Permissions

- Ecommerce is customer-facing; no staff permission roles apply to the storefront itself.
- Admin controls which products and designs are public (`is_public` flag).
- Admin configures the payment gateway credentials in Shop Settings.

---

## Dependencies

- Module 10 (Customers): customer account linking.
- Module 11 (Measurements): online customers can view/update their measurements.
- Module 15 (Product Catalog): `is_public` flag; prices.
- Module 16 (Roll Inventory): real-time stock availability.
- Module 23 (Stitching Orders): appointment bookings create orders.
- Module 24 (Design Gallery): `is_public` designs shown.
- Module 30 (Capacity): appointment slot availability.
- Module 37 (Notifications): confirmation and status notifications.
- Module 40 (Couriers): delivery tracking.

---

## Out of Scope (MVP)

- Customer reviews / ratings.
- Wishlist.
- Product recommendation engine.
- Loyalty points / referral programme.
- B2B wholesale ordering portal.
- Multiple storefronts per tenant (one storefront per tenant in MVP).
