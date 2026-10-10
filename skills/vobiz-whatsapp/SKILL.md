---
name: vobiz-whatsapp
description: Send and receive messages on the WhatsApp Business API via Vobiz - message templates, broadcast campaigns, inbound webhooks, multi-agent inbox, and BYON channel setup.
license: MIT
compatibility: Meta WhatsApp Business API via Vobiz as BSP.
metadata:
  author: vobiz
  version: "1.0"
---

# Vobiz WhatsApp skill

Use this when the user wants to **send WhatsApp messages programmatically, build a chatbot, run a campaign, or set up multi-agent customer support over WhatsApp**.

## How Vobiz fits

Vobiz is a Meta-approved BSP (Business Solution Provider). You connect your WhatsApp Business Account (WABA) once, then send/receive via Vobiz's REST API or no-code campaign builder.

Base URL for every messaging endpoint: `https://api.vobiz.ai/api/v1/messaging`. Auth on every request: `X-Auth-ID` (`MA_XXXXXXXX`) + `X-Auth-Token` headers (a login JWT via `Authorization: Bearer <token>` is also accepted). Send `Content-Type: application/json` with a body. Relative paths below are under that base URL.

## Capabilities

- **Messaging** - send `text`, `template`, and media (`image`, `audio`, `video`, `document`, `sticker`) via `POST /api/v1/messaging/messages`. Interactive (buttons, lists), `location`, and `contacts` are WhatsApp message types you **receive inbound**; the documented send contract covers the types above. To offer buttons outside the window, use an approved template with buttons.
- **Templates** - submit + manage Meta-approved templates, scoped per channel under `/channels/{channelId}/templates` (list with `?status=`, create, and `POST .../templates/sync`). Statuses: `APPROVED`, `PENDING_REVIEW`, `REJECTED`, `DISABLED`, `PAUSED`. Categories: `UTILITY`, `MARKETING`, `AUTHENTICATION`.
- **Campaigns** - `POST /campaigns` with `channel_id`, `name`, `template_id` (template **UUID**, not name), `template_components`, `audience_type` (`tags` + `audience_tags`, or `contact_ids` + `audience_contact_ids`), optional `recipient_variables`, `variable_mappings`, `scheduled_at` (omit to send now). Lifecycle `draft` -> `scheduled` -> `sending` -> `paused` -> `completed`/`cancelled`/`failed`; `POST /campaigns/{id}/pause|resume|cancel` (no body), `GET /campaigns/{id}/recipients` for per-contact status (`queued`/`sent`/`delivered`/`read`/`failed`/`skipped`).
- **Calls** - WhatsApp Business Calling over WebRTC (not PSTN). `POST /calls/whatsapp` with `channel_id`, `to`, `sdp_offer` returns `201` with `call_id` - the SDP answer arrives later via the `call.<event>` webhook, not in the response. `POST /calls/whatsapp/action` with `channel_id`, `call_id`, `action` (`pre_accept`/`accept`/`reject`/`terminate`; `sdp` required for `accept`/`pre_accept`). `GET /calls/whatsapp/logs` lists call history (`page`, `limit`, `direction`, `status`).
- **Inbox** - multi-agent dashboard for conversation handling; `GET /conversations` and `GET /conversations/{id}/messages` for history; `/contacts` CRUD + `POST /contacts/import`; `/canned-responses`.
- **Webhooks** - outbound events `message.inbound`, `message.status` (status = `sent`/`delivered`/`read`/`failed`), and `call.<event>` (`call.connect`, `call.status_change`, `call.terminate`); signed with `X-Webhook-Signature`, event name also in the `X-Webhook-Event` header. Manage subscriptions at `POST|GET /webhooks`, `DELETE /webhooks/{id}`.
- **Channels** - three onboarding modes: `bring_your_own` (BYON), `buy_from_vobiz`, `embedded_signup`. `POST|GET /channels/whatsapp`, `POST /channels/whatsapp/embedded-signup` (Meta OAuth `code`), `PUT|DELETE /channels/whatsapp/{id}`.
- **Numbers** - `GET /numbers/whatsapp/availability?country=IN` to search, `POST /numbers/whatsapp/orders` to buy a WhatsApp-capable number, or OTP-verify a BYON number before connecting.

## Core request shape (send a message)

Every send needs `channel_id` (UUID), `waba_id`, `to` (E.164 with leading `+`), and `type`. The send endpoint returns `201` with a `Message` object whose `status` starts at `pending` and is updated to `sent` -> `delivered` -> `read` (or `failed`) via `message.status` webhooks. The Meta id arrives as `meta_message_id` (the `wamid`) once Meta accepts it - `null` until then. A `2xx` means accepted, not delivered.

```json
{ "channel_id": "<uuid>", "waba_id": "<digits>", "to": "+919876543210",
  "type": "text", "text": { "body": "Hi!" } }
```

Media uses a single `media` object with either `link` (public HTTPS) **or** Meta media `id`, plus optional `caption`/`filename`. Templates use `type: "template"` with `template.name`, `template.language.code`, optional `category`, and `components[]` carrying ordered `parameters`.

## Key restrictions

- **24-hour customer-service window**: free-form messages (text/media) are allowed only for 24h after the customer's **last inbound** message. Each new inbound message resets the timer. Outside the window, you can send **only approved templates** - a free-form send is rejected by WhatsApp and surfaces as `failed` on the `message.status` webhook. Templates are always allowed; once the customer replies to one, a fresh 24h free-form window is open.
- **Templates** must be pre-approved by Meta. New templates start at `PENDING_REVIEW`; approval takes minutes to ~24h. After creating or approving templates in Meta Business Manager directly, call `POST /channels/{channelId}/templates/sync` to pull the latest statuses into Vobiz (returns `{ "synced": n }`).
- **One provider per number**: a WhatsApp number can be attached to exactly one provider (BSP) at a time. If it is live with another provider, deregister it there first; the number also cannot be registered on the WhatsApp or WhatsApp Business consumer app.
- **Quality rating + limits**: Meta assigns a Green/Yellow/Red quality rating and a daily messaging limit per number. Low quality (spam, blocks, slow replies) reduces messaging limits.

## Pitfalls

- **First message to a new contact must be a template.** A brand-new contact has never replied, so the 24h window is closed - a free-form `text` send fails. Use an approved template (often `UTILITY` or `MARKETING`) to open the conversation.
- **`delivered`/`read` are not events.** They are values of the `status` field inside a single `message.status` webhook. There is no `message.delivered`/`message.read`/`message.sent`/`message.failed`, nor `contact.updated`/`campaign.completed`. Only three outbound event types exist: `message.inbound`, `message.status`, `call.<event>`.
- **Verify `X-Webhook-Signature` over the raw bytes.** It is the hex HMAC-SHA256 of the exact request body keyed by your subscription `secret` - **no** `sha256=` prefix. Do not re-serialize parsed JSON before hashing; constant-time compare. (The Meta->Vobiz leg at `https://api.vobiz.ai/api/v1/webhooks/whatsapp` uses `X-Hub-Signature-256` with a `sha256=` prefix - different mechanism, you don't build against it.)
- **Two webhook systems.** Meta->Vobiz (platform plumbing, set once in your Meta App) is separate from Vobiz->your-server (what you build against). The `secret` is never returned after creation - store it when you create the subscription. The `wss://api.vobiz.ai/api/v1/messaging/ws` WebSocket is for live inbox UIs only, not a webhook substitute.
- **Template category mismatch = rejection.** Don't ship promotional content as `UTILITY`. `AUTHENTICATION` is OTP-only and gets the `COPY_CODE` button + expedited review.
- **`access_token` is write-only.** The Meta system-user token (starts with `EAA...`) is stored securely and never returned by any channel API. Rotate it via `PUT /channels/whatsapp/{id}` when it expires.
- **Deduplicate on `event_id`.** Deliveries are retried; the underlying Meta event can also repeat. Return `200` fast and process async.
- **Campaigns take a template UUID.** `template_id` is the Vobiz template `id` from `GET /channels/{channelId}/templates`, not the Meta template name.

## Recipes

- **Auto-reply to an inbound message**: receive `message.inbound` -> reply with `type: "text"` (you're inside the 24h window). See `whatsapp/webhooks/events` + `whatsapp/api/send-message`.
- **Proactive notification (window closed)**: create + get an approved template -> send with `type: "template"` and ordered `components[].parameters`. See `whatsapp/messaging/templates`.
- **Connect an existing number (BYON)**: `POST /numbers/whatsapp/bring-your-own/verify` with `phone_number` + `verification_method` (`sms`/`voice`) -> `POST /numbers/whatsapp/bring-your-own/confirm` with `phone_number` + `otp_code` -> `POST /channels/whatsapp` with `number_onboarding_mode: "bring_your_own"`, `waba_id`, `phone_number_id`, `phone_number`, `number_provider` (e.g. `meta_direct`), `access_token`, `display_name`. See `whatsapp/channels/byon` + `whatsapp/api/numbers`.
- **Track delivery**: store the returned `id`/`meta_message_id`, then correlate `message.status` webhooks by the `wamid` in `statuses[].id`.
- **Broadcast to a segment**: tag contacts -> `POST /campaigns` with `audience_type: "tags"` -> poll `GET /campaigns/{id}` counts or `GET /campaigns/{id}/recipients?status=failed`. See `whatsapp/api/campaigns`.

## When to search docs

- "Send a template message" -> `whatsapp/messaging/templates`, `whatsapp/api/send-message`
- "Inbound webhook / verify signature" -> `whatsapp/webhooks`, `whatsapp/webhooks/events`, `whatsapp/api/webhooks`
- "Connect my existing WhatsApp number (BYON)" -> `whatsapp/channels/byon`, `whatsapp/api/numbers`
- "Connect a channel via API / embedded signup" -> `whatsapp/api/channels`
- "Build a chatbot flow" -> `whatsapp/messaging`
- "Buy a WhatsApp number" -> `whatsapp/api/numbers`
- "Run a broadcast campaign" -> `whatsapp/api/campaigns`
- "WhatsApp voice calls" -> `whatsapp/api/calls`
- "24-hour window / conversation billing" -> `whatsapp/messaging`, `whatsapp/api/conversations`
