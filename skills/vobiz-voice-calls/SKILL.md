---
name: vobiz-voice-calls
description: Place, control, and inspect voice calls on Vobiz - make outbound calls, transfer, hangup, list live calls, and react to call events via webhooks.
license: MIT
compatibility: Vobiz REST API v1, X-Auth-ID + X-Auth-Token header auth.
metadata:
  author: vobiz
  version: "1.0"
---

# Vobiz Voice Calls skill

Use this when the user wants to **dial out, control an active call, or look up a live call**. For completed-call history, prefer the `vobiz-cdr` skill.

## What you can do

- **Make a call** - `POST /api/v1/Account/{auth_id}/Call/` with `{from, to, answer_url, answer_method}`. Returns `request_uuid` (same value as `call_uuid`) and `message: "Call fired"`.
- **List live calls** - `GET /api/v1/Account/{auth_id}/Call?status=live` returns an array of call UUIDs. `GET /api/v1/Account/{auth_id}/Call/?status=queued` lists queued (not yet connected) calls, max 20 per request.
- **Get one live call** - `GET /api/v1/Account/{auth_id}/Call/{call_uuid}?status=live` (or `?status=queued` for a pending call).
- **Hangup a call** - `DELETE /api/v1/Account/{auth_id}/Call/{call_uuid}` returns `204`. Also cancels a queued call.
- **Mid-call actions** - play audio, speak TTS, send DTMF, start/stop recording, start audio stream. Each is a `POST` to `/Call/{call_uuid}/Play/`, `/Speak/`, `/DTMF/`, `/Record/`, `/Stream/` (trailing slash).
- **Machine detection** - outbound calls support synchronous (`machine_detection=true|hangup`) and async (`machine_detection_url`) answering-machine detection with tunable timers. See `call/machine-detection`.

## Required inputs

- `from`: a Vobiz number you own (or a verified caller-ID), E.164.
- `to`: E.164 format (`+91...`) or a SIP URI. Use `<` (not a comma) to bulk-dial up to 1000 destinations in one request (e.g. `14157654321<14153464321<sip:agent@api.vobiz.ai`).
- `answer_url`: HTTPS endpoint that returns VobizXML when the callee answers.
- `answer_method`: HTTP verb for `answer_url`. The make-call schema lists it as required alongside `from`/`to`/`answer_url`; default to `POST`.
- Optional lifecycle webhooks (callbacks) on the same request: `ring_url`, `hangup_url`, `fallback_url` (+ `_method` for each), plus `time_limit` (seconds, default 14400), `caller_name`, `send_digits`. These extra params are documented in `call/make-call` but not in the OpenAPI request schema.

## Leg and casing semantics (memorize these)

- `legs`/`leg` selects who an in-call action hits: `aleg` = caller (the A-leg, your `from` side), `bleg` = callee, `both` = everyone. Default is `aleg` everywhere.
- **Play** and **Speak** use the plural field `legs` (`aleg`/`bleg`/`both`). **DTMF** uses the singular field `leg` (`aleg`/`bleg`/`both`). **Transfer** uses the plural `legs`.
- `request_uuid` and `call_uuid` are the same identifier returned by make-call; use either.
- The base host is `https://api.vobiz.ai`; every path is prefixed with `/api/v1`.

## Pitfalls

- Path casing matters: `/Account/.../Call/` (uppercase `Account`, uppercase `Call`) for call ops. Lowercase paths (`/account/.../call/`) return 401.
- Trailing slash matters and the two list routes differ by it in the OpenAPI spec: `GET /Call?status=live` lists live calls; `GET /Call/?status=queued` lists queued calls. `POST /Call/` (with slash) makes a call. When in doubt, copy the exact path from the matching doc page and always pass `status` explicitly.
- `status` is a **required** query param on every retrieve endpoint (`live` or `queued`). Omitting it returns the finalized CDR or a 404, not the live record.
- A `200` from make-call means the call was **accepted and queued**, not answered - watch `ring_url`/`answer_url`. The `answer_url` request arrives with `Event=StartApp` (not `Answer`); `hangup_url` gets `Event=Hangup` with `CallStatus=completed`.
- **Transfer is documented in `call/transfer-call` as `POST /Call/{call_uuid}/` and is not in `openapi.yaml`** (the spec only defines `GET` on that path for retrieve-queued). Body: `legs` (`aleg`|`bleg`|`both`, default `aleg`), `aleg_url`/`aleg_method`, `bleg_url`/`bleg_method`; the URLs must return VobizXML. Returns `202`. The call must be `in-progress`; queued or ended calls return `404`. In transfer XML that dials PSTN, set `<Dial callerId>` to a Vobiz number your account owns.
- **Conferences are created on join via XML**, not via REST. There is no "create conference" POST - a caller enters a `<Conference>NAME</Conference>` room and Vobiz auto-creates it. The REST Conference endpoints inspect/control existing rooms.
- Conference member IDs come from the `<Conference callbackUrl>` `enter` callback (`ConferenceMemberID`); capture them there rather than relying on `GET /Conference/{name}/`. `member_id` accepts a single id, a comma-separated list (`10,15,22`), or `all`. Member Mute/Deaf/Kick/Play return `202` with `member_id` as an array; Unmute/Undeaf/Stop/hang-up return `204`. Conference names with spaces must be URL-encoded (`My%20Conf%20Room`); names are case-sensitive.
- Play `urls`, Speak `text`, member Play `url` are required; sending them empty fails with `400`. Audio must be public `.mp3`/`.wav` over HTTP(S). Play `urls` takes a comma-separated string for sequential files.
- REST `Record/` defaults `time_limit` to **60 s** - raise it for long calls. `file_format` is `mp3`|`wav` (default `mp3`); `record_channel_type` is `mono`|`stereo`. Each `POST` starts a separate recording with its own `recording_id`.
- Stop actions are idempotent: stopping play/speak/recording when nothing is active still returns success (204). Hangup and conference deletes are destructive and not reversible.
- A real call costs money and can fail for **402 (insufficient balance)** or be throttled by **429 (CPS/concurrency limit)**. In sandbox/test work, place at most one short call per scenario.

## Flow recipes

- **Outbound call + answer flow:** `POST /Call/` with `{from,to,answer_url,answer_method}` → returns `request_uuid` and `message: "Call fired"`. On answer, Vobiz POSTs the standard call params (`CallUUID`, `From`, `To`, `Direction`, `CallStatus=in-progress`, `Event=StartApp`) to `answer_url`; reply with VobizXML. Configure `ring_url`/`hangup_url` for lifecycle webhooks (see `call/make-call`).
- **Voicemail handling:** add `machine_detection=true|hangup` (+ optional `machine_detection_url` for async). On async, Vobiz POSTs `Machine: true/false` and `Event=MachineDetection` to your URL while the call continues - branch on it. Keep an AI agent silent until the result arrives (see `call/machine-detection`).
- **Mid-call control:** retrieve the live `call_uuid`, then `POST /Call/{call_uuid}/Play|Speak|DTMF|Record/`. Pick `legs`/`leg` to target a side. Stop Play/Speak/Record with the matching `DELETE`.
- **Warm transfer:** `POST /Call/{call_uuid}/` with `legs=aleg` + `aleg_url` to send the caller to hold/another flow while you set up the callee, then transfer with new XML. The transfer target receives `Event=Transfer`.
- **Conference moderation:** caller joins via `<Conference callbackUrl="...">` XML → store `ConferenceMemberID` from the `enter` callback → `Mute`/`Deaf`/`Kick`/`Play` per member (or `all`) → record with a per-leg `<Record recordSession="true" redirect="false"/>` placed before `<Conference>` → `DELETE /Conference/{name}/` to end.

## When to search docs

- "How do I make a call from Python?" → search `call/make-call`
- "What does the answer URL receive?" → search `xml/request`
- "How do I detect voicemail?" → search `call/machine-detection`
- "How do I record a call?" → search `call/record-calls`
- "How do I transfer or redirect a live call?" → search `call/transfer-call`
- "How do I mute/kick a conference participant?" → search `conference/members`
- "How are conferences created?" → search `xml/conference`
