---
name: vobiz-applications-endpoints
description: Configure Vobiz Voice Applications and SIP Endpoints. Use when setting answer or hangup webhook URLs, attaching call-control applications to phone numbers, creating SIP credentials, registering softphones or desk phones, connecting browser/WebRTC clients, or troubleshooting endpoint registration and routing.
---

# Vobiz Applications & Endpoints skill

Two related but distinct concepts:

- **Application**: a named container that holds the `answer_url`/`hangup_url` webhooks. Phone numbers and endpoints get **attached** to an application so calls fire the configured webhooks.
- **Endpoint**: a SIP user account (`username` + `password` + `alias`) for soft-phones, browser WebRTC clients, or IP desk-phones to register with Vobiz.

Base URL: `https://api.vobiz.ai`. Auth headers on every request: `X-Auth-ID` + `X-Auth-Token`.

## Application vs Endpoint - which do I need?

- Inbound call to a **phone number (DID)** → create an **Application**, then **attach the number** to it.
- A **person or device** that needs to place/receive calls (browser, softphone, desk phone, AI agent) → create an **Endpoint**, then attach an Application to the endpoint for its call-handling logic.
- Both: number routes inbound to an Application; that Application's VobizXML can `Dial` the endpoint's `sip_uri` to ring the device.

## Application endpoints

| Op | Path | Notes |
|---|---|---|
| Create | `POST /api/v1/Account/{auth_id}/Application/` | Body: `{app_name, answer_url, answer_method, hangup_url?, hangup_method?, ...}`. Returns `app_id`. |
| List | `GET /api/v1/Account/{auth_id}/Application/` | Paginated: `limit` (default 20, max 100), `offset` (default 0). |
| Get | `GET /api/v1/Account/{auth_id}/Application/{app_id}/` | |
| Update | `POST /api/v1/Account/{auth_id}/Application/{app_id}/` | (yes, POST not PUT.) Partial update; omitted fields unchanged. |
| Delete | `DELETE /api/v1/Account/{auth_id}/Application/{app_id}/` | `204` on success; `409` if numbers are still attached. |

### Attach / detach a number (different path - under numbers, not Application)

| Op | Path |
|---|---|
| Attach | `POST /api/v1/Account/{auth_id}/numbers/{number}/application` body `{application_id}` |
| Detach | `DELETE /api/v1/Account/{auth_id}/numbers/{number}/application` |

`{number}` is the E.164 number **URL-encoded** (`+` → `%2B`, e.g. `%2B14155551234`).

## Endpoint endpoints

| Op | Path | Notes |
|---|---|---|
| Create | `POST /api/v1/Account/{auth_id}/Endpoint/` | Body: `{username, password, alias, application}` - `application` is the numeric `app_id`. Returns `endpoint_id` + `sip_uri`. |
| List | `GET /api/v1/Account/{auth_id}/Endpoint/` | Filters: `username__contains/__exact/__startswith`, `alias__contains/__exact`, `application_id__exact`, `application_id__isnull`, `sub_account`, plus `limit`/`offset`. |
| Get | `GET /api/v1/Account/{auth_id}/Endpoint/{endpoint_id}/` | Adds `sip_registration` block when `sip_registered == "true"`. |
| Update | `POST /api/v1/Account/{auth_id}/Endpoint/{endpoint_id}/` | (yes, POST not PUT.) `202` on success, empty body. |
| Delete | `DELETE /api/v1/Account/{auth_id}/Endpoint/{endpoint_id}/` | `204` on success. Disconnects registered devices immediately. |

## Default-app semantics

- `default_number_app: true` → any newly created number that has **no** `app_id` is auto-routed to this application.
- `default_endpoint_app: true` → any newly created endpoint with **no** `application` is auto-routed to this application.
- `default_app` is a separate read-mostly flag for the account-level default application.
- Setting a new default does not retroactively re-route numbers/endpoints created earlier - it only affects ones created afterward without an explicit app.

## Public SIP URI

- Each application exposes a `sip_uri` (e.g. `sip:<app_id>@sip.vobiz.ai`). `public_uri: false` by default.
- Set `public_uri: true` only when an external SIP system must reach the application's `sip_uri` directly (e.g. routing from a third-party PBX/carrier into your app). Leave it `false` for ordinary DID-driven inbound flows.

## Pitfalls

- **Path casing**: capital `Account` + capital `Application`/`Endpoint`. Lowercase segments 404. (Attach/detach is the exception: it uses lowercase `numbers`.)
- **Update is POST, not PUT/PATCH** - Vobiz convention for both Application and Endpoint.
- **SIP username**: documented constraint is alphanumeric only; treat `-`/`_`/spaces as risky (may 400). It is unique per account - a duplicate username returns a validation error (`409`/`400`). Username is **immutable** after creation; to rename, delete and recreate.
- **Locked endpoint fields**: `username`, `endpoint_id`, `domain`, `allow_same_domain`, `allow_other_domains`, `allow_phones`, `allow_apps` cannot be changed via update. To change them, recreate the endpoint.
- **Password is write-only** - never returned in any response. After a password change, every SIP client must re-register with the new credential or it drops.
- **`sip_registered` is a string** (`"true"`/`"false"`), not a boolean - compare against the string.
- **Deleting an app in use**: returns `409` if phone numbers are still attached. Detach all numbers first, then delete.
- **Attaching an already-linked number**: attaching overwrites the previous application binding (re-points routing); it does not error. Detach is idempotent-ish - detaching an unlinked number is a no-op/clears nothing.
- **Fallback**: `fallback_answer_url` fires only when `answer_url` is unreachable, times out, or returns invalid VobizXML. It must itself return valid XML. With no fallback set, the call drops on `answer_url` failure.
- **`hangup_url` defaults to `answer_url`** when omitted at create time - your answer handler may receive hangup callbacks unless you set a distinct `hangup_url`.
- **An endpoint binds to exactly one application** - swap it via update (`application` field).
- **WebRTC auth is SIP credentials, not JWT** - browser clients register with the endpoint `username`/`password`. There is no JWT-token API in v1.

## Recipes

### Create app → attach number → route inbound

1. `POST .../Application/` with `app_name`, `answer_url`, `answer_method` → capture `app_id`.
2. `POST .../numbers/%2B<E164>/application` with `{ "application_id": "<app_id>" }`.
3. Inbound calls to that number now fetch `answer_url` for VobizXML.
4. Verify with `GET .../Application/{app_id}/` and inspect the number's binding.

### Create endpoint → attach app → ring a device from a call

1. `POST .../Endpoint/` with `username`, `password`, `alias`, `application: <app_id>` → capture `endpoint_id` + `sip_uri`.
2. Register a softphone/browser/desk phone with the username/password against `sip.vobiz.ai`.
3. In the inbound Application's VobizXML, `Dial` the endpoint's `sip_uri` to ring the device.
4. Confirm registration via `GET .../Endpoint/{endpoint_id}/` → `sip_registered: "true"`.

### Rotate an endpoint password safely

1. `POST .../Endpoint/{endpoint_id}/` with new `password`.
2. Push the new credential to every SIP client and force re-registration (old credential stops working).

## When to search docs

- "How do I configure a Pipecat / LiveKit / Vapi / ElevenLabs handler?" → `integrations/*`
- "Browser calling with WebRTC" → `integrations/webrtc-application-setup`
- "Inbound call routing / DID binding" → `applications/attach-number`
- "VobizXML elements (Dial, Gather, etc.)" → voice XML reference docs
