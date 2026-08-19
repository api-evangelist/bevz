---
name: bevz-onboard-store
description: Register or provision a liquor/convenience store on the Bevz platform under an integrator account, configure it, and connect it to DoorDash or Grubhub delivery.
api: Bevz Integrator Service
version: 1.12.0
base_url: https://api.bevz.com/integrator-service
sandbox_url: https://sandbox-api.bevz.com/integrator-service
operations:
  - createStore
  - postStore
  - getStores
  - getIntegratorStore
  - patchStore
  - onboard
  - removeStore
generated: '2026-08-13'
method: generated
source: openapi/bevz-integrator-service-openapi.yaml
---

# Onboard a store on Bevz

Every operation below is grounded in a real `operationId` from the published Bevz Integrator Service
OpenAPI. All paths are nested under `/integrators/{integrator_id}` — the integrator is the tenancy
root and nothing is reachable without it.

## Before you start

- Mint a JWT: `POST {base_url}/integrators/login` with `{"email": ..., "password": ...}`. The token
  comes back at `data.token` and expires after **30 days**.
- Send `Authorization: Bearer <token>` on **every** request. The spec does not declare this as a
  security scheme, so a generated client will not add it for you — add it yourself.
- Work in sandbox (`https://sandbox-api.bevz.com/integrator-service`) until Bevz signs you off.

## Step 1 — decide: create or provision

Two different starting points, two different operations:

- The store does **not** exist on Bevz yet → `createStore`
  `POST /integrators/{integrator_id}/stores`
- The store **already** exists on Bevz (the merchant is already a Bevz customer) → `postStore`
  `POST /integrators/{integrator_id}/stores/provision`

Calling the wrong one is the most common failure. `postStore` returns `404` with
*"Cannot provision, store does not exist in Bevz"* when the store is not there, and `400` with
*"Cannot provision, store is already provisioned with another Bevz Integrator"* when someone else
owns it.

## Step 2 — create the store

`createStore` takes a `CreateStore` body with three parts: `store`, `account` and `operationHours`.

Validation is strict and field-level. The published `400` messages tell you exactly what failed:

- `"store.name" is not allowed to be empty`, `"store.address.city"/.state/.street1/.zipCode is not allowed to be empty`
- `"store.emailAddress" must be valid email`, `Invalid phone number.  Please input a valid phone number`
- `Invalid address. Please input a valid address` — the address is validated, not just present
- `"account.password" length must be at least 8 characters long`, `Password should be the same` (confirmPassword must match)
- `operationHours must be in hh:mm A format`, `Opening time should be before Closing time`
- `operationHours[0].days[1] must be greater than or equal to 1` / `less than or equal to 7` — days are 1–7
- `Email already exists. Please use a different email`

On success you get back a **store ID (UUID)**. Persist it; every later call needs it.

## Step 3 — verify

- `getStores` — `GET /integrators/{integrator_id}/stores` lists everything you have provisioned.
  Note this one returns a **bare array**, not the `{next_page, data}` envelope the other collections use.
- `getIntegratorStore` — `GET /integrators/{integrator_id}/stores/{store_id}` reads one store.

## Step 4 — configure the store

`patchStore` — `PATCH /integrators/{integrator_id}/stores/{store_id}` updates `name`,
`phoneNumber`, `address`, `hours`, `taxRate`, `enabled` and `deliverySettings`.

Guardrails from the published errors:

- `"enabled" is required` — send it.
- `unknown field in request body` — Bevz rejects unrecognized fields rather than ignoring them.
- `Doordash delivery settings is currently not configured. Please contact admin to assist you with onboarding.`
  — you cannot patch delivery settings for a service the store has not been onboarded to yet. Do
  step 5 first.

## Step 5 — connect a delivery service

`onboard` — `POST /integrators/{integrator_id}/stores/{store_id}/onboard-delivery-services`
covers **DoorDash and Grubhub**, setting commission rates, menu-sync options and enable/disable
status per service.

Two published `400` conditions gate DoorDash specifically: `doordash_hours` and
`doordash_inventory` — set store hours and stock the menu before onboarding.

**Uber Eats does not go through this operation.** It needs its own three-step OAuth handshake —
see the `bevz-onboard-ubereats` skill.

## Step 6 — subscription (optional)

- `generateCheckoutLink` — `GET .../generate-checkout-link` returns a subscription checkout link.
  It refuses when the store is already `active`, `trialing` or `cancelling`.
- `getSubscriptionLink` — `POST .../subscription-link` returns the subscription management link.
  (Yes, `POST` — it was changed from `GET` in release 1.10.2.)

## Deprovisioning

`removeStore` — `POST /integrators/{integrator_id}/stores/{store_id}/deprovision`. **Destructive.**

Published preconditions:
- `Cannot deprovision, store must be offline to continue deprovisioning` — disable it first via `patchStore`.
- `Cannot deprovision, store does not exist in Bevz` (404 since 1.10.7).
- `Cannot  deprovision, store is already provisioned with another Bevz Integrator`.
- `Store is currently not provisioned with any Integrator`.

## Rules that apply to every call

- **No idempotency.** There is no idempotency key anywhere in this API. `createStore` and `postStore`
  are non-idempotent POSTs — a blind retry after a timeout can double-create. Read back with
  `getStores` before retrying.
- **No rate limits are published**, and no `429` is documented. Be conservative anyway.
- **Errors are free text.** There are no error codes; you have to match on the `errors[]` strings.
- **401** means your JWT is missing, expired (>30 days) or invalid — re-login.
- **403** is the AWS API Gateway authorizer denying you outright, not a business error.
