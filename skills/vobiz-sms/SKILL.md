---
name: vobiz-sms
description: Send and receive SMS, MMS and RCS with the Vobiz Messaging API - single and multi-recipient sends, cost preview, idempotency, scheduling, delivery status, two-way conversations, opt-outs, webhooks, broadcasts, RCS senders and US 10DLC / India DLT compliance.
license: MIT
compatibility: Vobiz Messaging API v1 (https://api.vobiz.ai/api/v1/sms), X-Auth-ID + X-Auth-Token header auth.
metadata:
  author: vobiz
  version: "1.0"
---

# Vobiz SMS, MMS and RCS skill

Use this for **text messaging**: OTPs, notifications, two-way support threads, bulk broadcasts and rich RCS messages. For WhatsApp use the `vobiz-whatsapp` skill; for voice use `vobiz-voice-calls`.

Base URL: `https://api.vobiz.ai/api/v1/sms`. Auth headers on every request: `X-Auth-ID` and `X-Auth-Token`. JSON fields are **camelCase** (`mediaUrls`, `scheduledAt`, `idempotencyKey`), unlike the snake_case Voice API.

## Before the first send

- **A sender.** `from` is a messaging-capable Vobiz number in E.164 for `sms`/`mms` (list them with `GET /numbers`), or the `senderName` of an approved RCS sender for `rcs`.
- **Compliance.** US long codes need a registered 10DLC brand and campaign with the number attached (`/10dlc/...`); US toll-free numbers need a toll-free campaign; India traffic needs TRAI DLT IDs (`dltEntityId`, `dltHeaderId`, `dltTemplateId`) on every message and `text` matching the registered template.
- **Consent.** Only message people who opted in. Vobiz records `STOP`-style replies as opt-outs and refuses later sends to them.

## Send a message

`POST /messages`

| Field | Notes |
|---|---|
| `from` | Required. E.164 number (sms/mms) or RCS `senderName` (rcs) |
| `to` | Required. Array of E.164 numbers; each recipient becomes its own message and is billed separately; blanks and duplicates are skipped |
| `channel` | `sms` (default), `mms` or `rcs`. An `sms` with `mediaUrls` to US/Canada recipients is sent as MMS |
| `text` | Required for sms and rcs |
| `mediaUrls` | Public `https` URLs. MMS up to 10; RCS sends the first as a file |
| `title` | MMS subject |
| `tag` | Your label, stored on every message the request creates; filter `GET /messages?tag=` by it |
| `idempotencyKey` | Your unique key per recipient; retrying with the same key returns the original message instead of sending again |
| `scheduledAt` | RFC 3339 time, e.g. `2026-11-01T09:00:00Z`; the message is held with status `scheduled` |
| `failoverToSms` | RCS only: fall back to SMS with the same `text` |
| `rcsSuggestions` | RCS only: reply and action buttons |

Responses: `201` sent (the created message), `202` accepted but the message failed (read `errorCode` / `errorMessage`), `400` validation error, `422` recipient opted out (do not retry), `502` the request could not be processed. The body is **one message object**: for multi-recipient sends, set a `tag` and list by it to get every message ID, or send one request per recipient with its own `idempotencyKey`.

## Segments and cost

GSM-7 text fits 160 characters in one segment (153 per segment when split); any non-GSM character (emoji, most non-Latin scripts, curly quotes) switches the whole message to UCS-2: 70 characters (67 per segment). `GET /messages/cost-preview?from=&to=&text=&channel=` returns `segmentCount`, `unitCost`, `totalCost`, `currency` and `country` without sending or billing anything. Preview before any bulk or long send.

## Status and history

Statuses: `scheduled` → `sent` (handed to the network) → `delivered` or `failed`; inbound messages are `received`; `unknown` means the network reported a state outside this list (read the message history). `direction` is `out` or `in`.

- `GET /messages` - filters `status`, `channel`, `direction`, `to`, `from`, `tag`, `startDate`, `endDate`; paging `limit` (1-100, default 20) and `offset`; returns `{messages, total, limit, offset}`.
- `GET /messages/{id}` - one message with its status history.

## Conversations (two-way)

Every message joins a conversation per number pair and channel.

- `GET /conversations` - newest activity first, with `messageCount` and `unreadCount`.
- `GET /conversations/{id}/messages` - the thread.
- `POST /conversations/{id}/read` - mark read; `GET /conversations/unread-count` - total unread.

Reply by sending from the number the contact wrote to.

## Opt-outs

| Op | Path |
|---|---|
| Add | `POST /opt-outs` `{phoneNumber, channel?, reason?}` - an `sms` opt-out also blocks `mms` |
| List | `GET /opt-outs?limit=&offset=` - pass both |
| Check | `GET /opt-outs/{phoneNumber}?channel=sms` → `{optedOut}` |
| Remove | `DELETE /opt-outs/{phoneNumber}?channel=sms` - only after the person opted back in (e.g. replied `START`) |

URL-encode the `+` in path parameters (`%2B14155550123`).

## Webhooks

`POST /webhooks` `{url, secret, events}` subscribes an https URL. Default events: `message.received`, `message.delivered`, `message.failed`. Verify the `X-Webhook-Signature` HMAC with your secret and de-duplicate on `X-Webhook-ID`. These are separate from voice callbacks (`X-Vobiz-Signature-*`).

## Broadcasts (bulk)

1. `POST /broadcasts` `{name, from, channel, template, ratePerSecond?, scheduledAt?, tag?}` - `template` uses `{{variable}}` placeholders.
2. `POST /broadcasts/{id}/recipients` - JSON `{numbers, defaultCountry}` or a CSV whose columns fill the variables.
3. `GET /broadcasts/{id}/preview` - rendered samples, `encoding`, `segments` and `missingVariables`. Fix anything listed before launching.
4. `POST /broadcasts/{id}/launch`; control with `/pause`, `/resume`, `/cancel`.
5. `GET /broadcasts/{id}` for `stats`; `GET /broadcasts/{id}/recipients` and `/export` (CSV) for per-recipient results. Opted-out recipients are skipped automatically.

## RCS

Register a sender with `POST /senders`, check `GET /senders/{id}/status`, and manage templates under `/senders/{senderId}/templates`. Send with `channel: "rcs"` and `from` set to the sender's `senderName`; use `failoverToSms: true` so contacts without RCS still get the text.

## Pitfalls

- Use E.164 everywhere (`+14155550123`); local formats are rejected or mis-routed.
- Every `POST /messages` sends a real, billed message. Confirm sender, recipients and text with the user first, and always pass an `idempotencyKey` so retries never double-send.
- `422` means opted out: respect it, do not retry or switch channel to get around it.
- A `202` is not success; check `status` and `errorMessage` on the returned message.
- `sent` is not `delivered`. Follow `GET /messages/{id}` or the `message.delivered` / `message.failed` webhooks.
- Ignore the `provider` field: it is an internal routing identifier.
- Plan templates for UCS-2 if they can contain emoji or non-Latin names; a single character can triple the segment count.

## When to search docs

Search the Vobiz docs (or the `vobiz-docs` MCP server) for: the full field list of any endpoint, 10DLC brand and campaign registration, toll-free verification, India DLT, RCS sender onboarding, webhook payload examples, and per-country coverage (`GET /coverage`).
