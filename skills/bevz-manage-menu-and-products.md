---
name: bevz-manage-menu-and-products
description: Upload a store menu to Bevz as CSV or JSON, track the asynchronous upload through the Menu Upload Webhook, sync it to the delivery services, and maintain per-product pricing and availability.
api: Bevz Integrator Service
version: 1.12.0
base_url: https://api.bevz.com/integrator-service
operations:
  - postMenu
  - getMenu
  - menuSync
  - getProducts
  - patchProduct
  - deleteProduct
  - patchMenuUpload
generated: '2026-08-13'
method: generated
source: openapi/bevz-integrator-service-openapi.yaml
---

# Manage a Bevz store menu and its products

Menu upload is **asynchronous**. You submit a file, you get an upload id back, and the result
arrives on a webhook. Treat it as a job, not a request/response.

## Step 0 — register the menu webhook first

`patchMenuUpload` — `PATCH /integrators/{integrator_id}` with `menuUploadNotificationUrl`.

This is flagged **Required** on the Bevz integration checklist. Without it you have no completion
signal and must poll.

> **Known contract defect.** In the published spec this operation sits on a path key ending in an
> invisible `U+3164 HANGUL FILLER` character, added to avoid a duplicate-key collision with
> `patchOrder`. The real path is `/integrators/{integrator_id}`. A generated client will
> percent-encode the invisible character and get a 404 — hand-write this call.

Published `400`s: `"menuUploadNotificationUrl" is required`, `"menuUploadNotificationUrl" must be a string`.

## Step 1 — upload the menu

`postMenu` — `POST /integrators/{integrator_id}/stores/{store_id}/menu`

- Accepts **CSV or JSON**. `Incorrect file format! File should be in CSV or JSON format.` is the
  rejection.
- Required per item since 1.10.7: `upc`, `name`, `price`. (`size` and `quantity` used to be required
  and no longer are.)
- Optional `merchantSuppliedId` carries *your* product id so you can map Bevz products back to your
  own catalog. Set it — it comes back on order line items later and is the only mapping key you get.
- Column names are configurable for CSV via `upc_column_name`, `price_column_name`,
  `name_column_name` and `stockcount_column_name`.

**One upload per store at a time.** A concurrent upload is rejected with
`An ongoing upload for this storeId is still in progress, please wait for upload to complete`.
This lock is the only throttling behavior Bevz publishes — respect it and serialize per store.

You get back an `uploadMenuId` (a UUID since 1.10.0 — it used to be an integer).

## Step 2 — track the upload

Two ways, use both:

- **Webhook** (preferred): the Menu Upload Webhook fires as the upload moves through in-progress,
  completed and failed. The payload carries `id`, `store_id`, `status`, `time_completed` and —
  critically — `error_file`, a link to the rows that failed.
- **Poll**: `getMenu` — `GET /integrators/{integrator_id}/stores/{store_id}/menu/{menu_id}`.

**Always read `error_file` on completion.** A menu upload can "complete" with rows rejected; if you
only check status you will silently ship a partial menu.

## Step 3 — sync to the delivery services

`menuSync` — `POST /integrators/{integrator_id}/stores/{store_id}/menu-sync` pushes the menu out to
the connected delivery services.

Published `400`s and what they mean:
- `Delivery settings not configured` — onboard the store to a delivery service first.
- `Service disabled on store` — the service is connected but switched off.
- `Not authorized to use service` — this integrator is not entitled to that service.

## Step 4 — maintain products

- `getProducts` — `GET .../products`. **Paginated**: send `limit`, then follow the `next_page` token
  from the response. Returns `{next_page, data}`. Products carry `productId`, `upc`,
  `merchantSuppliedId`, `name`, `size`, `quantity`, `categoryL1`/`categoryL2`, `images`, `prices`,
  `inStorePrice`, `forSale` and `isAvailableOnDoorDash`.
- `patchProduct` — `PATCH .../products/{product_id}` updates price, stock and availability.
  - `inStorePrice must be a positive number`
  - `stockCount must be greater than or equal to 0`
  - `Product ID is not found under this store`
  - `"randomfield" is not allowed` — unknown fields are rejected, not ignored.
  - **Watch out:** setting `stock` to `0` inside `pricingOverride` *deletes the product variant*
    (behavior added in 1.10.1). If you mean "out of stock", that is not the same thing as "delete".
- `deleteProduct` — `DELETE .../products/{product_id}`. **Destructive.**
  `Unable to delete item, store product does not exist` on a miss.

## Rules

- No idempotency key exists on any of these operations. Re-uploading a menu is not a safe retry —
  check `getMenu` first.
- No rate limits are published. The per-store upload lock is your only backpressure signal.
- The per-service price fields (`bevzPrice`, `doordashPrice`, `grubhubPrice`, `uberEatsPrice`) live
  under `pricingOverride` on `getProducts` — read them before overwriting a price.
