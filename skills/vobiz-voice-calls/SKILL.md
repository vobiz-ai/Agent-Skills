---
name: vobiz-voice-calls
description: Place, control, and inspect live Vobiz voice calls. Use when creating outbound calls, selecting answer URLs, transferring or hanging up active calls, listing or retrieving live calls, handling call-status webhooks, diagnosing call initiation errors, or deciding when completed-call work belongs in the CDR workflow instead.
---

# Vobiz Voice Calls skill

Use this when the user wants to **dial out, control an active call, or look up a live call**. For completed-call history, prefer the `vobiz-cdr` skill.

## What you can do

- **Make a call** - `POST /api/v1/Account/{auth_id}/Call/` with `{from, to, answer_url, answer_method}`. Returns a `call_uuid`.
- **List live calls** - `GET /api/v1/Account/{auth_id}/Call` to see all in-progress/queued calls.
- **Get one live call** - `GET /api/v1/Account/{auth_id}/Call/{call_uuid}`.
- **Hangup a call** - `DELETE /api/v1/Account/{auth_id}/Call/{call_uuid}`.
- **Mid-call actions** - play audio, speak TTS, send DTMF, start/stop recording, start audio stream. Each is a `POST` to `/Call/{call_uuid}/{action}/`.
- **Machine detection** - outbound calls support synchronous and async answering-machine detection with configurable sensitivity. See `call/machine-detection`.

## Required inputs

- `from`: a Vobiz number you own (or a verified caller-ID).
- `to`: E.164 format (`+91...`). Use `<` to bulk-dial up to 1000 destinations in one request (e.g. `14157654321<14153464321<sip:agent@api.vobiz.ai`).
- `answer_url`: HTTPS endpoint that returns VobizXML when the callee answers.
- `answer_method`: HTTP verb for `answer_url`. The make-call schema treats this as required alongside `from`/`to`/`answer_url`; default to `POST`.

## Leg and casing semantics (memorize these)

- `legs`/`leg` selects who an in-call action hits: `aleg` = caller (the A-leg, your `from` side), `bleg` = callee, `both` = everyone. Default is `aleg` everywhere.
- **Play** and **Speak** use the plural field `legs` (`aleg`/`bleg`/`both`). **DTMF** uses the singular field `leg` and accepts `aleg`/`bleg`/`both` per the API schema (some examples show only `aleg`/`bleg`).
- `request_uuid` and `call_uuid` are the same identifier returned by make-call; use either.
- The base host is `https://api.vobiz.ai`; every path is prefixed with `/api/v1`.

## Pitfalls

- Path casing matters: `/Account/.../Call/` (uppercase `Account`, uppercase `Call`) for call ops. Lowercase paths (`/account/.../call/`) return 401.
- Trailing slash matters and the two list routes differ by it: `GET /Call?status=live` lists live calls; `GET /Call/?status=queued` lists queued calls. `POST /Call/` (with slash) makes a call. When in doubt, copy the exact path from the matching doc page.
- `status` is a **required** query param on every retrieve endpoint (`live` or `queued`). Omitting it returns the finalized CDR or a 404, not the live record.
- **Transfer is NOT a REST endpoint in the OpenAPI spec** - it is `POST /Call/{call_uuid}/` documented in `call/transfer-call`. Don't confuse it with `GET /Call/{call_uuid}/` (retrieve queued). The transfer body uses `aleg_url`/`bleg_url` and the plural `legs` selector; the new URLs must return VobizXML.
- **Conferences are created on join via XML**, not via REST. There is no "create conference" POST - a caller enters a `<Conference>NAME</Conference>` room and Vobiz auto-creates it. The REST Conference endpoints only inspect/control/record existing rooms.
- `member_id` accepts a single id, a comma-separated list (`10,15,22`), or `all`. Conference names with spaces must be URL-encoded (`My%20Conf%20Room`); names are case-sensitive.
- Play `urls`, Speak `text`, member Play `url` are required; sending them empty fails. Audio must be public `.mp3`/`.wav` over HTTP(S).
- Stop actions are idempotent: stopping play/speak/recording when nothing is active still returns success (204). Hangup and conference deletes are destructive and not reversible.
- A real call costs money and can fail for **402 (insufficient balance)** or be throttled by **429 (CPS/concurrency limit)**. In sandbox/test work, place at most one short call per scenario.

## Flow recipes

- **Outbound call + answer flow:** `POST /Call/` with `{from,to,answer_url,answer_method}` → returns `request_uuid` and `message: "Call fired"`. On answer, Vobiz POSTs the answer-URL params to `answer_url`; reply with VobizXML. Configure `ring_url`/`hangup_url` for lifecycle webhooks (see `call/make-call`).
- **Voicemail handling:** add `machine_detection=true|hangup` (+ optional `machine_detection_url` for async). On async, Vobiz POSTs `Machine: true/false` to your URL while the call continues - branch on it (see `call/machine-detection`).
- **Mid-call control:** retrieve the live `call_uuid`, then `POST /Call/{call_uuid}/Play|Speak|DTMF|Record/`. Pick `legs`/`leg` to target a side. Stop with the matching `DELETE`.
- **Warm transfer:** `POST /Call/{call_uuid}/` with `legs=aleg` + `aleg_url` to send the caller to hold/another flow while you set up the callee, then transfer with new XML.
- **Conference moderation:** caller joins via `<Conference>` XML → `GET /Conference/{name}/` for `member_id`s → `Mute`/`Deaf`/`Kick`/`Play` per member (or `all`) → `Record` the room → `DELETE` to end.

## When to search docs

- "How do I make a call from Python?" → search `call/make-call`
- "What does the answer URL receive?" → search `xml/request`
- "How do I detect voicemail?" → search `call/machine-detection`
- "How do I record a call?" → search `call/record-calls`
- "How do I transfer or redirect a live call?" → search `call/transfer-call`
- "How do I mute/kick a conference participant?" → search `conference/members`
- "How are conferences created?" → search `xml/conference`
