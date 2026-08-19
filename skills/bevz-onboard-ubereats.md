---
name: bevz-onboard-ubereats
description: Run the three-step Uber Eats OAuth onboarding handshake through the Bevz Integrator Service — generate the merchant authorization URL, exchange the returned code for tokens, and link the Bevz store to its Uber Eats counterpart.
api: Bevz Integrator Service
version: 1.12.0
base_url: https://api.bevz.com/integrator-service
operations:
  - generateOAuth
  - exchangeCode
  - provisionStore
  - onboard
  - patchStore
generated: '2026-08-13'
method: generated
source: openapi/bevz-integrator-service-openapi.yaml
---

# Onboard a store to Uber Eats through Bevz

Uber Eats is the **exception** in the Bevz delivery-onboarding model. DoorDash and Grubhub go through
the single `onboard` operation. Uber Eats needs a three-step OAuth authorization-code handshake with
a human in the middle, because the store owner must personally grant access to their Uber Eats account.

Added in release 1.11.0.

## Prerequisites

- The store exists and is provisioned (see the `bevz-onboard-store` skill).
- You hold a valid Bevz JWT — `Authorization: Bearer <token>` on every call.
- Your `redirect_uri` is **already registered with Uber Eats**. The Bevz call will accept a URI that
  Uber Eats does not know, and the failure will surface later as an OAuth error.

## Step 1 — generate the authorization URL

`generateOAuth` — `GET /integrators/{integrator_id}/stores/{store_id}/onboard-delivery-services/ubereats/generate-oauth`

Required query parameter: `redirect_uri` — must match the URI registered with Uber Eats.

Returns a unique OAuth URL. **Send the store owner to it.** They authenticate with Uber Eats and
authorize your integration. Uber Eats then redirects back to your `redirect_uri` with an
authorization `code` on the query string.

This operation declares only `200`, `401` and `403` — no `400`. Do not expect Bevz to validate the
redirect URI for you.

## Step 2 — exchange the code

`exchangeCode` — `POST .../onboard-delivery-services/ubereats/exchange-authorization-code`

Send the authorization code you caught on the redirect. Bevz exchanges it with Uber Eats and returns
the access token material plus the Uber Eats store identifier.

The `400` responses here are **standard OAuth 2.0 error codes passed through from Uber Eats** — this
is the one place in the whole Bevz contract where you get machine-readable error codes:

| code | what went wrong |
|---|---|
| `invalid_request` | The exchange request was malformed. |
| `invalid_client` | Client credentials rejected by Uber Eats. |
| `invalid_grant` | The code is expired, already used, or does not match the redirect URI. |
| `invalid_scope` | The requested scope was not granted. |
| `access_denied` | The store owner declined authorization. |

`invalid_grant` is the common one: authorization codes are single-use and short-lived. If you retry
step 2, you must go back to step 1 and get a fresh code — there is no idempotency here and replaying
a code will always fail.

## Step 3 — provision the store on Uber Eats

`provisionStore` — `POST .../onboard-delivery-services/ubereats/provision-store`

Send the access token and the Uber Eats store ID from step 2. This links the Bevz store to its Uber
Eats counterpart and switches on menu synchronization and order management.

Published `400`s:
- `Invalid OAuth 2.0 credentials provided.` — the token from step 2 is wrong, expired, or for a
  different store.
- `User not allowed to access the store` — the authorizing account does not own that Uber Eats store.
- `Invalid request` — malformed body.

## Step 4 — verify and configure

- `patchStore` — set commission rates and enable/disable state under `deliverySettings`. Trying this
  before onboarding completes returns the *"delivery settings is currently not configured"* class of
  error.
- Then run `menuSync` (see the `bevz-manage-menu-and-products` skill) to push the menu to Uber Eats.
- Then confirm order flow with `testStoreOrders`.

## What this flow moves

The delivery-service onboarding schemas (`DeliverySettingsUE`, `SensitiveDataUE`) carry
**merchant-of-record data**: legal business name, legal representative name, legal email, legal date
of birth, EIN, SSN, bank account number, routing number and entity type.

Bevz publishes no field-level handling, masking or retention policy for any of it, no security
program, and no compliance certification. Handle this payload accordingly on your side: do not log
it, do not cache it, and do not put it through an agent that retains context.

## Rules

- **No idempotency.** Every step is a non-idempotent POST and step 2 is actively single-use.
- **No rate limits published.**
- Uber Eats order acceptance is **manual** and the window is **10 minutes** — unlike DoorDash and
  Grubhub, nothing auto-accepts. See the `bevz-process-orders` skill before you go live.
