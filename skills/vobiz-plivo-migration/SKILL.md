---
name: vobiz-plivo-migration
description: Migrate an existing Plivo voice integration to Vobiz with minimal code changes. Use when replacing Plivo SDK authentication and base URLs, converting PlivoXML to VobizXML, updating webhook signature handling, validating equivalent call APIs, or provisioning replacement Vobiz numbers because Plivo numbers cannot be ported.
---

# Vobiz Plivo migration skill

Use this when porting an existing **Plivo** voice app to **Vobiz**. The two platforms are near 1:1: the SDK method names match, `calls.create(...)` has the same signature, the `/Account/{auth_id}/Call/` path is identical, and VobizXML ≈ PlivoXML. **Prefer the smallest possible diff** - do not rewrite working logic.

## Migration at a glance

For most apps the entire migration is:

1. Swap the package/client (`plivo` → `vobiz`) and credentials/base URL.
2. Rename `<GetDigits>` / `<GetInput>` → `<Gather>` in your XML (and `timeout`→`executionTimeout`, `type`→`inputType`).
3. Point the webhook signature check at `X-Vobiz-Signature-V2/V3` (same HMAC-SHA256+nonce scheme).
4. Buy **new Vobiz numbers** and re-point your flows - you can't port Plivo numbers.
5. Re-point `answer_url` / `hangup_url`, test, canary, cut over.

## What actually changes

| Layer | Plivo | Vobiz |
|---|---|---|
| Package / client | `import plivo` · `plivo.RestClient(auth_id, auth_token)` | `import vobiz` · `vobiz.RestClient(auth_id, auth_token)` |
| Env vars | `PLIVO_AUTH_ID` / `PLIVO_AUTH_TOKEN` | `VOBIZ_AUTH_ID` / `VOBIZ_AUTH_TOKEN` |
| Raw-HTTP auth | Basic `auth_id:auth_token` | two headers `X-Auth-ID` + `X-Auth-Token` |
| Base URL | `https://api.plivo.com/v1` | `https://api.vobiz.ai/api/v1` |
| XML input verb | `<GetDigits>` / `<GetInput>` | `<Gather>` (`type`→`inputType`, `timeout`→`executionTimeout`) |
| XML builder | `add_get_digits` / `add_get_input` | `add_gather` |
| Webhook signature | `X-Plivo-Signature-V3` | `X-Vobiz-Signature-V2` / `X-Vobiz-Signature-V3` |
| App SIP domain | `sip.plivo.com` | `app.vobiz.ai` / `<trunk>.sip.vobiz.ai` |
| Phone numbers | your existing Plivo DIDs | **buy new Vobiz numbers - no porting** |

## What does NOT change (don't "fix" it)

- `client.calls.create(from_, to_, answer_url, answer_method, ring_url, hangup_url, fallback_url, machine_detection, sip_headers, time_limit, ...)` - same signature; only the client init differs. Response: `request_uuid` (== `call_uuid`), `api_id`, `message` (Vobiz returns "Call fired").
- **Bulk dial uses the `<` separator on both** (`to_="a<b<c"`). There were never commas - do not change it.
- Path casing + trailing slash (`/Account/.../Call/`), machine-detection params, and the webhook signature **scheme** (HMAC-SHA256 + nonce) are all the same.
- All other XML verbs map 1:1: `Speak`, `Play`, `Dial` (+`Number`/`User`), `Record`, `Conference`, `Redirect`, `Hangup`, `Wait`, `PreAnswer`, `DTMF`.

## Endpoint & SDK mapping

Method names match; only client init changes. Endpoints sit under `/api/v1/Account/{auth_id}/...`.

| Action | SDK (both) | Vobiz REST |
|---|---|---|
| Make a call | `client.calls.create(...)` | `POST /Account/{auth_id}/Call/` |
| Get / list live calls | `client.calls.get` / `list` | `GET /Account/{auth_id}/Call/{uuid}?status=live` · `GET /Account/{auth_id}/Call?status=live` |
| Hang up | `client.calls.delete` | `DELETE /Account/{auth_id}/Call/{uuid}` |
| Applications | `client.applications.*` | `…/Application/` (+ attach/detach `Number`) |
| Recordings | `client.recordings.*` | `…/Recording/` |
| Conferences | `client.conferences.*` | `…/Conference/` (created on join via XML) |
| Sub-accounts | `client.subaccounts.*` | `…/SubAccount/` |
| Numbers | `client.numbers.*` | inventory + `…/numbers/purchase-from-inventory` |

## PlivoXML → VobizXML

The only real rename: `<GetDigits>` and `<GetInput>` → `<Gather>`. Inside it, `type`→`inputType` and `timeout`→`executionTimeout`. In the builder, `add_get_digits(...)`/`add_get_input(...)` → `add_gather(...)` on `vobiz.vobizxml.ResponseElement()`. Every other verb maps 1:1 by name and nesting. Return the document as `Content-Type: application/xml`.

## Phone numbers - you can't port from Plivo

**There is no number-porting/transfer flow from Plivo (or any provider) into Vobiz.** Plivo numbers stay on Plivo. To run on Vobiz, buy new numbers from inventory (`GET …/inventory/numbers` → `POST …/numbers/purchase-from-inventory` with `{e164}`), route them, and update every place the old number is published (app, CRM, website, IVR prompts, outbound caller ID). Keep Plivo numbers live during cutover, then release them on Plivo. Never tell a user to "port" their Plivo number.

## Webhook signature (header swap, not a rewrite)

Plivo's `validate_v3_signature(method, uri, nonce, auth_token, params, v3_signature)` is HMAC-SHA256 over `baseURL + "." + nonce` with header `X-Plivo-Signature-V3`. Vobiz uses the same scheme: `X-Vobiz-Signature-V2` (HMAC-SHA256 of `baseURL + nonce`) and `X-Vobiz-Signature-V3` (`baseURL + "." + nonce`), keyed by `auth_token`, base64-encoded, with a nonce header. Swap the header name and use the Vobiz validator. For sub-accounts, Vobiz also sends `X-Vobiz-Signature-MA-V2/V3` keyed by the **parent** token. (Only legacy Plivo V1 was HMAC-SHA1.) Canonical: `/concepts/validating-callbacks`.

## Pitfalls

- **`<Gather>` uses `executionTimeout`, never `timeout`.** `<GetDigits timeout="5">` → `<Gather executionTimeout="5">`; `type` → `inputType`.
- **Path casing + trailing slash matter.** `/Account/{auth_id}/Call/` is PascalCase with a trailing slash; lowercase or a missing slash returns 401.
- **`status` is a required query param** on call-retrieve endpoints (`live`/`queued`); omitting it returns the finalized CDR or 404.
- **Signature is a header swap, not a crypto rewrite.** Move from `X-Plivo-Signature-V3` to `X-Vobiz-Signature-V2/V3`; same HMAC-SHA256+nonce.
- **You can't port numbers from Plivo.** Buy new Vobiz numbers; never suggest a port/transfer.
- **Update hardcoded SIP URIs** from `sip.plivo.com` to `app.vobiz.ai` / `<trunk>.sip.vobiz.ai`.
- **DID 15-day cool-off:** unassigning a recently-used number from a sub-account returns `409 did_cool_off_in_effect` (admin `?force=true` override).
- **Buying a number debits `setup_fee + monthly_fee` immediately** (parent `MA_` charged for `SA_` purchases); a failed debit surfaces as `500`, an unavailable number as `404`.
- `<MultiPartyCall>` and other advanced Plivo verbs may have no direct Vobiz equivalent - flag for review, don't invent a mapping.

## Python recipe (before → after)

```python
# BEFORE (Plivo)
import plivo
client = plivo.RestClient("AUTH_ID", "AUTH_TOKEN")
client.calls.create(from_="+14155550100", to_="+14155550111",
                    answer_url="https://example.com/answer", answer_method="POST")

# AFTER (Vobiz) - only the import + client change
import vobiz
client = vobiz.RestClient("MA_XXXXXXXX", "AUTH_TOKEN")
client.calls.create(from_="+14155550100", to_="+14155550111",
                    answer_url="https://example.com/answer", answer_method="POST")
```

XML builder: `response.add_get_digits(...)` / `response.add_get_input(...)` → `response.add_gather(action=..., input_type=..., execution_timeout=...)` on `vobiz.vobizxml.ResponseElement()`.

## Node recipe (before → after)

```js
// BEFORE (Plivo)
const plivo = require('plivo');
const client = new plivo.Client('AUTH_ID', 'AUTH_TOKEN');
await client.calls.create('+14155550100', '+14155550111', 'https://example.com/answer', { answerMethod: 'POST' });

// AFTER (Vobiz)
const { Client } = require('vobiz-node');
const client = new Client('MA_XXXXXXXX', 'AUTH_TOKEN');
await client.calls.create('+14155550100', '+14155550111', 'https://example.com/answer', { answerMethod: 'POST' });
```

## Drop-in shim (coming soon)

A future `vobiz-plivo-compat` (Python) / `@vobiz/plivo-compat` (Node) package will expose the `plivo` surface but target Vobiz, so existing Plivo code runs with one import change. Treat as planned - do not claim it ships today.

## When to search docs

- "Full migration steps" → `/guides/plivo-to-vobiz`
- "Convert PlivoXML interactively" → `/migrate-plivo-to-vobiz` (tool)
- "Buy a Vobiz number (can't port from Plivo)" → `/buy-a-phone-number`, `/guides/plivo-to-vobiz/number-porting`
- "Gather attributes" → `/xml/gather`
- "Verify the webhook signature" → `/concepts/validating-callbacks`
- "Make-call params" → `/call/make-call`
