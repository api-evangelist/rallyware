---
name: Authenticate against a Rallyware tenant and page a collection
description: >-
  Obtain an OAuth2 access token from a Rallyware tenant, keep it fresh across a 401,
  and walk a Hydra paginated collection to completion. Every Rallyware integration
  starts here — there is no other way in.
api: mcp/rallyware-mcp.yml
generated: '2026-08-14'
method: generated
source: >-
  npm @rallyware/sdk-react-native-components@1.2.1 — lib/module/auth/*.js,
  lib/module/services/rallyware-api-service.js, lib/module/models/collection-response.js
operations:
  - getPublicConfig
  - getCurrentUser
  - getMyProfile
  - listBadges
  - listBadgeAchievers
  - listMyTaskPrograms
---

# Authenticate against a Rallyware tenant and page a collection

Rallyware publishes no OpenAPI and no API reference. Every operation, path, header
and response field below is read from **Rallyware's own published React Native SDK**
(`@rallyware/sdk-react-native-components@1.2.1`, npm). Do not invent paths or
parameters beyond these — nothing else about this API is public.

## Before you start

**You need a tenant host.** There is no shared `api.rallyware.com`. Rallyware runs one
deployment per customer, on either `https://{tenant}.rallyware.com` or a
customer-branded domain (Rallyware's own README shows both
`https://feature-frontend.rallyware.com` and `https://testlearningcenter.nuskin.com`).
The host is a required input. If you do not have one, stop — you cannot discover it.

**You need end-user credentials.** The only published grants are the OAuth2
resource-owner **password** grant and **refresh_token**. There is no
`client_credentials` grant, so there is no service-to-service token: you are acting as
a named human user, with that user's data visibility. Treat this as a red flag, not a
convenience — never hold these credentials in an autonomous loop without an explicit
human mandate.

## Step 1 — (optional) read the tenant bootstrap, unauthenticated

`getPublicConfig` → `GET {host}/api/public/config`

No token required. Returns `config.parameters` (a tenant settings map) and
`config.assets.wysiwyg`. The tenant's legal pages are addressed by well-known keys:

- `core.security.static_page.privacy_policy`
- `core.security.static_page.cookie_policy`
- `core.security.static_page.terms_of_service`

Use this to confirm the host is a live Rallyware tenant before spending a credential.

## Step 2 — get an access token

`POST {host}/oauth/v2/token`

```
grant_type=password
username={login}
password={password}
```

The response carries `access_token`, `refresh_token`, and two account-state flags:
`is_activated` and `is_email_verified`.

**Check both flags.** Rallyware's own client treats the API as unavailable unless
`is_activated && is_email_verified` are true, even with a valid token. If either is
false, stop and report that the account is not provisioned — do not retry.

## Step 3 — set the required headers on every call

```
Authorization: Bearer {access_token}
Rallyware-Data-Client-Device-Platform: SDK
Rallyware-Data-SDK-Version: 1.2.1
```

Also send `_locale={langCode}` as a **query parameter on every request** — Rallyware
negotiates language by query param, not by `Accept-Language`. Default `en`.

Do **not** send cookies. Rallyware's own SDK sets `withCredentials: false` with the
comment that cookies break the auth flow.

## Step 4 — handle the 401 refresh, exactly once

On any `401`:

1. `POST {host}/oauth/v2/token` with `grant_type=refresh_token` and your `refresh_token`.
2. Replay the original request **once**, with the new bearer token.
3. If the refresh itself fails, clear all auth state and surface the error. Do not loop.

Guard the replay with a per-request flag. Rallyware's client uses a `_retry` boolean
for exactly this reason. `401` is the only HTTP status with defined client behaviour
in this API — see `errors/rallyware-problem-types.yml`.

## Step 5 — page a collection

Collections are **Hydra** (JSON-LD) documents. Every list operation
(`listBadges`, `listBadgeAchievers`, `listMyTaskPrograms`) returns:

| Field | Meaning |
|---|---|
| `hydra:member` | the items on this page |
| `hydra:view.hydra:next` | URL of the next page, **absent on the last page** |
| `hydra:totalItems` | total across all pages |

Loop: request the collection, consume `hydra:member`, then follow
`hydra:view.hydra:next` until it is absent. Pass `items_per_page` to size the page
(Rallyware's client defaults to `1000`).

Never compute the next page yourself from an offset — follow the link. The absence of
`hydra:next` is the only reliable end-of-collection signal.

## Step 6 — identify the caller

- `getCurrentUser` → `GET /api/users/me` — identity, level, custom attributes, points.
- `getMyProfile` → `GET /api/users/me/profile` — **returns PII (`email`)** plus KPI
  values and `is_rules_agreed`. Gate this behind an explicit need; do not fetch it to
  populate a display name that `/api/users/me` already provides.

To resolve a user from the customer's own system of record, use
`getUserByExternalId` → `GET {host}/users/external/{id}`. Note the missing `/api`
prefix — that is how Rallyware's SDK calls it, verbatim.

## What you cannot rely on

- **No rate limits are published** and Rallyware's own client has no `429` handling or
  backoff. Add your own conservative pacing and exponential backoff.
- **No error schema is published.** Only `401` has defined semantics. Treat any other
  non-2xx as opaque: log the body, do not parse it, do not branch on it.
- **No idempotency.** No `Idempotency-Key` is supported on any write. See the
  companion skill before retrying a state-changing call.
