---
name: vobiz-plivo-migration
description: Migrate a Plivo voice integration to Vobiz - swap the client/auth and base URL, rename one XML verb plus a few attributes, update the webhook signature header, and provision new Vobiz numbers from the Vobiz inventory (existing Plivo numbers stay with Plivo). Plivo and Vobiz are near 1:1, so prefer minimal diffs.
license: MIT
compatibility: Vobiz REST API v1 (X-Auth-ID + X-Auth-Token), VobizXML 1.0; mirrors Plivo v1 + PlivoXML.
metadata:
  author: vobiz
  version: "1.1"
---

# Vobiz Plivo migration skill

Use this when porting an existing **Plivo** voice app to **Vobiz**. The two platforms are near 1:1: the `/Account/{auth_id}/Call/` path is identical, the call parameters (`from`, `to`, `answer_url`, ...) carry over, and VobizXML mirrors PlivoXML. What changes is mechanical: the SDK client + a few method names, the auth headers and base URL, one XML verb, the webhook signature header, and the phone numbers. **Prefer the smallest possible diff** - do not rewrite working logic.

## Migration at a glance

For most apps the entire migration is:

1. Swap the package/client (`plivo.RestClient(...)` -> `Vobiz(api_key=, auth_token=)` / `new VobizClient({ apiKey, authToken })`), credentials, and base URL.
2. Rename `calls.create(...)` -> `calls.make_call(...)` (Python) / `calls.makeCall(...)` (Node), pass `auth_id` explicitly, `to_` -> `to`, and always pass `answer_method`.
3. Rename `<GetDigits>` / `<GetInput>` -> `<Gather>` in your XML (`type` -> `inputType`, `timeout` -> `executionTimeout`, `digitTimeout` -> `digitEndTimeout`).
4. Point the webhook signature check at `X-Vobiz-Signature-V2` / `X-Vobiz-Signature-V3` using the Vobiz validator (HMAC-SHA256 of base URL + nonce).
5. Buy **new Vobiz numbers** and re-point your flows. Vobiz provisions new numbers from its own inventory; existing Plivo numbers stay with Plivo.
6. Re-point `answer_url` / `hangup_url`, test in sandbox, canary, cut over.

## What actually changes

| Layer | Plivo | Vobiz |
|---|---|---|
| Package / client | `import plivo` · `plivo.RestClient(auth_id, auth_token)` | Python `from vobiz import Vobiz` · `Vobiz(api_key=AUTH_ID, auth_token=AUTH_TOKEN)`; Node `import { VobizClient } from '@vobiz/sdk'` · `new VobizClient({ apiKey, authToken })`. `api_key` **is** the Auth ID (`MA_...`/`SA_...`) |
| SDK install | package registries | clone `github.com/vobiz-ai/Vobiz-<Language>-SDK` and install from the checkout (Go: `go get github.com/vobiz-ai/Vobiz-Go-SDK`); same 7 languages |
| Make a call | `client.calls.create(from_=, to_=, answer_url=, ...)` | `client.calls.make_call(auth_id=, from_=, to=, answer_url=, answer_method=, ...)` - explicit `auth_id`, `to_` -> `to`, `answer_method` required |
| Env vars | `PLIVO_AUTH_ID` / `PLIVO_AUTH_TOKEN` | `VOBIZ_AUTH_ID` / `VOBIZ_AUTH_TOKEN` |
| Raw-HTTP auth | HTTP Basic `auth_id:auth_token` | two headers `X-Auth-ID` + `X-Auth-Token` |
| Base URL | `https://api.plivo.com/v1` | `https://api.vobiz.ai/api/v1` (note the extra `/api`) |
| XML input verb | `<GetDigits>` / `<GetInput>` | `<Gather>` (`type` -> `inputType`, `timeout` -> `executionTimeout`, `digitTimeout` -> `digitEndTimeout`) |
| XML builder | `plivoxml.ResponseElement()` · `add_get_digits` / `add_get_input` | `vobizxml.ResponseElement()` · `add_gather` (`add_get_digits`/`add_get_input` kept as aliases) |
| Webhook signature | `X-Plivo-Signature-V3` (+ `-Nonce`) | `X-Vobiz-Signature-V2` / `X-Vobiz-Signature-V3` (+ `-Nonce`); Vobiz validator |
| App SIP domain | `sip.plivo.com` | `app.vobiz.ai` (applications) / `<trunk>.sip.vobiz.ai` (trunks) |
| Phone numbers | your existing Plivo DIDs | **buy new Vobiz numbers - no porting** |

## What does NOT change (don't "fix" it)

- Call **parameters**: `from`, `to`, `answer_url`, `answer_method`, `ring_url`, `hangup_url`, `fallback_url`, `machine_detection`, `sip_headers`, `time_limit` - pass them exactly as on Plivo. Response: `api_id`, `request_uuid` (== `call_uuid`), `message: "Call fired"`. A `200` means accepted and queued, not answered.
- **Bulk dial uses the `<` separator on both** (`to="a<b<c"`, up to 1000 destinations). There were never commas - do not change it.
- Path casing + trailing slash (`/Account/.../Call/`) and the callback **params** (`CallUUID`, `From`, `To`, `Direction`, `CallStatus`, ...) delivered as `application/x-www-form-urlencoded`.
- All other XML verbs map 1:1: `Speak`, `Play`, `Dial` (+`Number`/`User`), `Record`, `Conference`, `Redirect`, `Hangup`, `Wait`, `PreAnswer`, `DTMF`, `Stream`.

## Endpoint & SDK mapping

Endpoints sit under `https://api.vobiz.ai/api/v1/Account/{auth_id}/...`. SDK names below are Python; Node uses the same names in camelCase (`make_call` -> `makeCall`, `live_calls` -> `liveCalls`). Every account-scoped method takes `auth_id` explicitly.

| Action | Plivo SDK | Vobiz SDK (Python) | Vobiz REST |
|---|---|---|---|
| Make a call | `client.calls.create(...)` | `client.calls.make_call(auth_id, from_, to, answer_url, answer_method, ...)` | `POST /Account/{auth_id}/Call/` |
| Get / list live calls | `client.calls.get` / `list` | `client.live_calls.get_live_call(auth_id, call_uuid, status='live')` / `list_live_calls(auth_id, status='live')` (queued: `get_queued_call` / `list_queued_calls`) | `GET /Account/{auth_id}/Call/{uuid}?status=live` · `GET /Account/{auth_id}/Call?status=live` |
| Hang up | `client.calls.hangup` / `delete` | `client.live_calls.hangup_call(auth_id, call_uuid)` | `DELETE /Account/{auth_id}/Call/{uuid}` |
| Play / speak / DTMF on a live call | `client.calls.play` / `speak` / `send_digits` | `client.play_audio.call(...)`, `client.speak_text.call(...)`, `client.dtmf.send_dtmf(...)` | `POST .../Call/{uuid}/Play/` · `/Speak/` · `/DTMF/` |
| Record a live call | `client.calls.record` / `record_stop` | `client.record_calls.start_recording(...)` / `stop_recording(...)` | `POST|DELETE .../Call/{uuid}/Record/` |
| Transfer a live call | `client.calls.transfer` | return `<Redirect>` / `<Dial>` from your answer URL | (VobizXML) |
| Applications | `client.applications.*` | `client.applications.create_application / list_applications / retrieve_application / update_application / delete_application` | `.../Application/` (+ attach/detach `Number` pages) |
| Recordings | `client.recordings.*` | `client.recordings.list_recordings / get_recording / delete_recording` | `.../Recording/` |
| Conferences | `client.conferences.*` | `client.conferences.list_conferences / get_conference / delete_conference`; members: `client.conference.kick_member / hangup_member / deaf_member / play_audio_member`, `client.conference_members.mute_member / unmute_member`; `client.conference_recording.start_conference_recording / stop_conference_recording` | `.../Conference/{name}/...` (rooms are created on join via XML) |
| Sub-accounts | `client.subaccounts.*` | `client.sub_accounts.create_subaccount / list_subaccounts / retrieve_subaccount / update_subaccount / delete_subaccount` | `.../accounts/{auth_id}/sub-accounts/` (per `openapi.yaml`) |
| Numbers | `client.numbers.search / buy / list / delete` | `client.phone_numbers.list_inventory_numbers / purchase_from_inventory / list_numbers / unrent_number` | `GET .../inventory/numbers` -> `POST .../numbers/purchase-from-inventory` |
| Completed calls | `client.calls.list` | `client.cdr.list_cdrs(auth_id, start_date=, end_date=)` | `GET .../cdr` |

## PlivoXML -> VobizXML

The only real rename: `<GetDigits>` and `<GetInput>` -> `<Gather>`. Inside it, `type` -> `inputType` (`dtmf`, `speech`, `dtmf speech`), `timeout` -> `executionTimeout` (5-60s, default 15), `digitTimeout` -> `digitEndTimeout`. A `<GetInput>` that already uses `inputType`/`executionTimeout` is a tag rename only. In the builder, `add_get_digits(...)`/`add_get_input(...)` -> `add_gather(action=..., method=..., input_type=..., num_digits=..., execution_timeout=...)` on `vobizxml.ResponseElement()` (`from vobiz import vobizxml`). Every other verb maps 1:1 by name and nesting. Two attribute details: `<Conference maxMembers>` -> `maxParticipants`, and give every `<Record>` an `action` URL so Vobiz delivers `RecordUrl`. Return the document as `Content-Type: application/xml`.

## Phone numbers - provision new Vobiz numbers

**Vobiz provisions new numbers from its own inventory; existing Plivo numbers stay with Plivo.** To run on Vobiz, buy new numbers from inventory (`GET .../inventory/numbers?country=IN` -> `POST .../numbers/purchase-from-inventory` with `{ "e164": "+91..." }`, leading `+` included), route them (attach to an application or assign to a trunk), and update every place the old number is published (app, CRM, website, IVR prompts, outbound caller ID). Keep Plivo numbers live during cutover, then release them on Plivo. Guide users to new Vobiz numbers rather than a port.

## Webhook signature (same family, use the Vobiz validator)

Both platforms sign callbacks with HMAC-SHA256 keyed by your auth token, base64-encoded, with a random nonce in a companion header. Vobiz signs the **base URL with query stripped + nonce**: `X-Vobiz-Signature-V3` = `base64(HMAC-SHA256(authToken, baseURL + "." + nonce))` with `X-Vobiz-Signature-V3-Nonce`; `X-Vobiz-Signature-V2` = `base64(HMAC-SHA256(authToken, baseURL + nonce))` with `X-Vobiz-Signature-V2-Nonce`. Replace Plivo's `validate_v3_signature(...)` helper with the few-line Vobiz validator (it does not hash the POST params) and compare with `hmac.compare_digest` / `crypto.timingSafeEqual`. For sub-accounts, Vobiz also sends `X-Vobiz-Signature-MA-V2/V3` keyed by the **parent** token - same algorithm, different key. The legacy `X-Vobiz-Signature` (and Plivo's legacy `X-Plivo-Signature-V2`) are HMAC-SHA1 - avoid both. Signature headers are emitted only when the callback URL has auth credentials configured on it, so configure them and fail closed if an expected header is absent. Branch on the `Event` field (`Ring`, `StartApp` = answered, `Hangup`). Canonical: https://www.vobiz.ai/docs/concepts/validating-callbacks.

## Pitfalls

- **`<Gather>` uses `executionTimeout`, never `timeout`.** `<GetDigits timeout="5">` -> `<Gather executionTimeout="5">`; `type` -> `inputType`; `digitTimeout` -> `digitEndTimeout`. In VobizXML `timeout` belongs only to `<Dial>`/`<Number>` (ring timeout); a `timeout` left on `<Gather>` is ignored.
- **Path casing + trailing slash matter.** `/Account/{auth_id}/Call/` is PascalCase with a trailing slash; lowercase or a missing slash returns 401.
- **`status` is a required query param** on call-retrieve endpoints (`live`/`queued`); omitting it returns the finalized CDR or 404.
- **`answer_method` is required on `make_call`.** Plivo defaults it; Vobiz expects it passed explicitly.
- **`to_` -> `to`** on `make_call` (keep `from_`). A silent rename that breaks if missed.
- **Signature: swap the header and the validator.** Move from `X-Plivo-Signature-V3` to `X-Vobiz-Signature-V2/V3`; same HMAC-SHA256 + nonce family, signed string is base URL + nonce.
- **Numbers come from Vobiz inventory.** Buy new Vobiz numbers; existing Plivo numbers stay with Plivo, so suggest new numbers rather than a port/transfer.
- **Update hardcoded SIP URIs** from `sip.plivo.com` to `app.vobiz.ai` / `<trunk>.sip.vobiz.ai`.
- **DID 15-day cool-off:** unassigning a recently-used number from a sub-account returns `409 did_cool_off_in_effect` (with `cool_off_until`, `cool_off_remaining_seconds`); admin `?force=true` override writes an audit record.
- **Buying a number debits `setup_fee + monthly_fee` immediately** (parent `MA_` charged for `SA_` purchases); a failed debit surfaces as `500` (number stays in inventory), an unavailable number as `404`.
- **`<MultiPartyCall>` and other advanced Plivo verbs:** express them with Vobiz `<Conference>` (XML) plus the Conference API, and confirm the flow with Vobiz support before migrating - don't invent a one-to-one mapping.

## Python recipe (before -> after)

```python
# BEFORE (Plivo)
import plivo
client = plivo.RestClient("AUTH_ID", "AUTH_TOKEN")
client.calls.create(from_="+14155550100", to_="+14155550111",
                    answer_url="https://example.com/answer", answer_method="POST")

# AFTER (Vobiz) - new constructor, make_call, explicit auth_id, to_ -> to
from vobiz import Vobiz
client = Vobiz(api_key="MA_XXXXXXXX", auth_token="YOUR_AUTH_TOKEN")
client.calls.make_call(auth_id="MA_XXXXXXXX",
                       from_="+14155550100", to="+14155550111",
                       answer_url="https://example.com/answer", answer_method="POST")
```

XML builder: `response.add_get_digits(...)` / `response.add_get_input(...)` -> `response.add_gather(action=..., method=..., input_type="dtmf", num_digits=..., execution_timeout=...)` on `vobizxml.ResponseElement()` (`from vobiz import vobizxml`), then `response.to_string()`.

## Node recipe (before -> after)

```js
// BEFORE (Plivo)
const plivo = require('plivo');
const client = new plivo.Client('AUTH_ID', 'AUTH_TOKEN');
await client.calls.create('+14155550100', '+14155550111', 'https://example.com/answer', { answerMethod: 'POST' });

// AFTER (Vobiz)
import { VobizClient } from '@vobiz/sdk';
const client = new VobizClient({ apiKey: 'MA_XXXXXXXX', authToken: 'YOUR_AUTH_TOKEN' });
await client.calls.makeCall({
  auth_id: 'MA_XXXXXXXX',
  from: '+14155550100', to: '+14155550111',
  answer_url: 'https://example.com/answer', answer_method: 'POST',
});
```

XML builder: `import { vobizxml } from '@vobiz/sdk'`; `new vobizxml.ResponseElement()` -> `addGather({ action, method, inputType: 'dtmf', numDigits })` -> `toString()`.

## Compatibility package (planned)

A `vobiz-plivo-compat` (Python) / `@vobiz/plivo-compat` (Node) package that exposes the `plivo` surface but targets Vobiz is a planned addition. It is not published, so migrate with the Vobiz SDK as shown above and do not tell users to install it.

## When to search docs

- "Full migration steps" -> https://www.vobiz.ai/docs/guides/plivo-to-vobiz (sub-pages: auth-and-base-url, endpoint-mapping, xml-migration, webhooks-and-signatures, gotchas, number-porting, cutover-checklist)
- "Side-by-side Plivo vs Vobiz code" -> https://www.vobiz.ai/docs/compare/plivo/code-snippets; per-domain tables under https://www.vobiz.ai/docs/compare/plivo/overview
- "Let an AI agent convert my Plivo code" -> https://www.vobiz.ai/docs/compare/plivo/migration-agent
- "Buy a Vobiz number for a migrated flow" -> https://www.vobiz.ai/docs/buy-a-phone-number, https://www.vobiz.ai/docs/guides/plivo-to-vobiz/number-porting
- "Gather attributes" -> https://www.vobiz.ai/docs/xml/gather
- "Verify the webhook signature" -> https://www.vobiz.ai/docs/concepts/validating-callbacks
- "Make-call params" -> https://www.vobiz.ai/docs/call/make-call
