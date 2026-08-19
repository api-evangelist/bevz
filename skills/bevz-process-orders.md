---
name: bevz-process-orders
description: Receive delivery orders from DoorDash, Grubhub and Uber Eats through the Bevz Order Notification Webhook, drive them through the order lifecycle, and apply post-acceptance item adjustments within each delivery service's rules.
api: Bevz Integrator Service
version: 1.12.0
base_url: https://api.bevz.com/integrator-service
operations:
  - patchOrder
  - getStoreOrders
  - getOrderId
  - patchOrderStatus
  - postOrderAdjustment
  - testStoreOrders
generated: '2026-08-13'
method: generated
source: openapi/bevz-integrator-service-openapi.yaml
---

# Process Bevz delivery orders

Orders arrive from DoorDash, Grubhub and Uber Eats and are normalised by Bevz into one order object.
This is a **push** integration: you register a webhook and react. Polling is the fallback, not the design.

## Step 0 — register the order webhook

`patchOrder` — `PATCH /integrators/{integrator_id}` with `orderNotificationUrl`.

**Required** on the Bevz integration checklist. Published `400`s:
`"orderNotificationUrl" is required`, `"orderNotificationUrl" must be a string`.

> Same invisible-character caveat as `patchMenuUpload`: the two operations share the real path
> `/integrators/{integrator_id}` and the spec disambiguates one of them with a `U+3164` filler.

## Step 1 — consume the Order Notification Webhook

Bevz POSTs `{type, data}` to your URL. The `type` values are:

| type | meaning |
|---|---|
| `order.pending` | New order created, awaiting acceptance. **Act on this one.** |
| `order.accepted` | Accepted by the store. |
| `order.out_for_delivery` | Picked up by the courier. |
| `order.completed` | Delivered. |
| `order.rejected` | Store could not fulfill. |
| `order.canceled` | Canceled by customer or store. |

> **Verify nothing, because there is nothing to verify with.** Bevz publishes no webhook signature,
> shared secret or timestamp scheme, and no retry/dedupe/ordering guarantees. Treat the payload as
> untrusted: use it as a *trigger*, then read the order back with `getOrderId` before acting on
> money or inventory. Make your handler idempotent on `orderId` yourself — Bevz will not do it for you.

## Step 2 — read the order

- `getOrderId` — `GET /integrators/{integrator_id}/stores/{store_id}/orders/{order_id}`.
  `Invalid orderId!` on a malformed id.
- `getStoreOrders` — `GET .../orders` for a list. **Paginated** with `limit` + `next_page`; supports
  `date_field` plus date-range filtering (added 1.11.1).

An order carries `orderId`, `storeId`, `integratorId`, `customerId`, `deliverySource`, `orderStatus`,
`orderProducts[]` (each with a point-in-time `productSnapshot` and your `merchantSuppliedId`),
`deliveryDetails`, `fees`, `orderTotal` and `createdAt`.

## Step 3 — drive the status

`patchOrderStatus` — `PATCH .../orders/{order_id}` with `{"status": "..."}`.

Valid statuses: `ACCEPTED`, `OUT_FOR_DELIVERY`, `COMPLETED`, `CANCELED`, `REJECTED`.

**Which transitions you must make by hand differs per delivery service** — this is the part that
breaks integrations:

| | DoorDash | Uber Eats | Grubhub |
|---|---|---|---|
| ACCEPTED | automatic — auto-accepts after 1 min; confirm within 3–8 min | **manual, confirm within 10 min** | automatic — auto-accepts after 1 min; confirm within 10 min |
| OUT_FOR_DELIVERY | automatic | automatic | automatic |
| COMPLETED | automatic | automatic | automatic |
| REJECTED | **manual** — reason required | **manual** | **manual** — reason required |
| CANCELED | manual after acceptance — reason required | manual after acceptance | manual after acceptance — reason required |

Uber Eats is the one that will bite you: nothing auto-accepts, and the window is 10 minutes.

Published `400`s: `Invalid order status`, `Unable to update order status from incorrect state`
(transitions are state-machine checked), `Store is disabled, unable to perform this action`.

## Step 4 — adjust an order after acceptance

`postOrderAdjustment` — `POST .../orders/{order_id}/adjustments`.

Types: `ITEM_UPDATE`, `ITEM_REMOVE`, `ITEM_SUBSTITUTE`.

Per-service rules, all enforced and all published as `400`s:

- **Grubhub**: `ITEM_UPDATE` and `ITEM_REMOVE` only — no `ITEM_SUBSTITUTE`. **One adjustment per order.**
- **Uber Eats**: `UE - UberEats only allows an item to be replaced once`;
  `UE - UberEats exceeded the allowed quantity`;
  `UE - Product quantity must be less than of the current quantity`.
- **DoorDash**: `DD - Doordash Order Adjustment cannot be empty`.
- `name` is required in the `adjustments` object for `ITEM_SUBSTITUTE` (added 1.11.3).

Preconditions: `Order must be on Active/Accepted state`, `Order Products cannot be empty`,
`Product is not for sale`.

## Step 5 — test it before you go live

`testStoreOrders` — `POST /integrators/{integrator_id}/stores/{store_id}/orders` creates a **mock
order** against a provisioned store so you can exercise the whole lifecycle in sandbox. Accepts
`delivery_source`, `delivery_type`, `order_items`, `delivery_address` and `delivery_instructions`.
`order_items.product_id` accepts either a UUID or a UPC.

The Bevz certification test cases you must pass: create a test order, receive order notifications,
accept, cancel, mark out for delivery, and complete.

## Rules

- **No idempotency key** on `patchOrderStatus` or `postOrderAdjustment`. A retried status transition
  can hit `Unable to update order status from incorrect state` — read the order first.
- **No rate limits published**, no `429` documented.
- **No error codes** — branch on the free-text `errors[]` strings, and expect them to change without
  notice (they are not versioned and the changelog does not track them).
- Clock discipline matters more than usual here: acceptance windows are 3–10 minutes depending on the
  service, and Bevz publishes no webhook retry policy, so a missed delivery is a lost order.
