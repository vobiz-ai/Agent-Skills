---
name: vobiz-whatsapp
description: Build WhatsApp Business messaging workflows through Vobiz. Use when sending approved templates or session messages, receiving inbound webhook events, creating broadcast campaigns, configuring BYON channels, managing contacts or conversations, building chatbots, or setting up a multi-agent customer-support inbox.
---

# Vobiz WhatsApp skill

Use this when the user wants to **send WhatsApp messages programmatically, build a chatbot, run a campaign, or set up multi-agent customer support over WhatsApp**.

## How Vobiz fits

Vobiz is a Meta-approved BSP (Business Solution Provider). You connect your WhatsApp Business Account (WABA) once, then send/receive via Vobiz's REST API or no-code campaign builder.

## Capabilities

- **Messaging** - send `text`, `template`, and media (`image`, `audio`, `video`, `document`, `sticker`) via `POST /api/v1/messaging/messages`. Interactive (buttons, lists), `location`, and `contacts` are WhatsApp message types you **receive inbound**; the documented send contract covers the types above.
- **Templates** - submit + manage Meta-approved templates, scoped per channel under `/channels/{channelId}/templates`
- **Inbox** - multi-agent dashboard for conversation handling
- **Webhooks** - outbound events `message.inbound`, `message.status` (status = `sent`/`delivered`/`read`/`failed`), and `call.<event>`; signed with `X-Webhook-Signature`
- **Channels** - three onboarding modes: `bring_your_own` (BYON), `buy_from_vobiz`, `embedded_signup`
- **Numbers** - search/buy WhatsApp-capable numbers, or OTP-verify a BYON number before connecting

## Core request shape (send a message)

Every send needs `channel_id` (UUID), `waba_id`, `to` (E.164 with leading `+`), and `type`. The send endpoint returns `201` with a `Message` object whose `status` starts at `pending` and is updated to `sent` → `delivered` → `read` (or `failed`) via `message.status` webhooks. The Meta id arrives as `meta_message_id` (the `wamid`) once Meta accepts it — `null` until then.

```json
{ "channel_id": "<uuid>", "waba_id": "<digits>", "to": "+919876543210",
  "type": "text", "text": { "body": "Hi!" } }
```

Media uses a single `media` object with either `link` (public HTTPS) **or** Meta media `id`, plus optional `caption`/`filename`. Templates use `type: "template"` with `template.name`, `template.language.code`, and `components[]` carrying ordered `parameters`.

## Key restrictions

- **24-hour customer-service window**: free-form messages (text/media/interactive) are allowed only for 24h after the customer's **last inbound** message. Each new inbound message resets the timer. Outside the window, you can send **only approved templates** — a free-form send will be rejected. Sending a template opens a fresh 24h window.
- **Templates** must be pre-approved by Meta. New templates start at `PENDING_REVIEW`; approval takes minutes to ~24h. After approving (or editing) in Meta Business Manager directly, call `POST /channels/{channelId}/templates/sync` to pull the new status into Vobiz.
- **One BSP per number**: a WhatsApp number can be on exactly one BSP at a time. Moving from Twilio/Wati/etc. requires deregistering it there first; the number cannot be live on the WhatsApp/WhatsApp Business consumer app.
- **Quality rating + tiers**: Meta assigns a Green/Yellow/Red quality rating and a daily messaging limit per number. Low quality or high block/report rates reduce limits and can pause templates.

## Pitfalls

- **First message to a new contact must be a template.** A brand-new contact has never replied, so the 24h window is closed — a free-form `text` send fails. Use an approved template (often `UTILITY` or `MARKETING`) to open the conversation.
- **`delivered`/`read` are not events.** They are values of the `status` field inside a single `message.status` webhook. There is no `message.delivered`/`message.read`/`message.sent`/`message.failed`, nor `contact.updated`/`campaign.completed`. Only three outbound event types exist: `message.inbound`, `message.status`, `call.<event>`.
- **Verify `X-Webhook-Signature` over the raw bytes.** It is the hex HMAC-SHA256 of the exact request body keyed by your subscription `secret` — **no** `sha256=` prefix. Do not re-serialize parsed JSON before hashing. (The Meta→Vobiz leg uses `X-Hub-Signature-256` with a `sha256=` prefix — different mechanism, you don't build against it.)
- **Two webhook systems.** Meta→Vobiz (platform plumbing, set once in your Meta App) is separate from Vobiz→your-server (what you build against). The `secret` is never returned after creation — store it when you create the subscription.
- **Template category mismatch = rejection.** Don't ship promotional content as `UTILITY`. `AUTHENTICATION` is OTP-only and gets the `COPY_CODE` button + expedited review.
- **`access_token` is write-only.** The Meta system-user token (starts with `EAA…`) is stored securely and never returned by any channel API. Rotate it via `PUT /channels/whatsapp/{id}` when it expires.
- **Deduplicate on `event_id`.** Deliveries are retried; the underlying Meta event can also repeat. Return `200` fast and process async.

## Recipes

- **Auto-reply to an inbound message**: receive `message.inbound` → reply with `type: "text"` (you're inside the 24h window). See `whatsapp/webhooks/events` + `whatsapp/api/send-message`.
- **Proactive notification (window closed)**: create + get an approved template → send with `type: "template"` and ordered `components[].parameters`. See `whatsapp/messaging/templates`.
- **Connect an existing number (BYON)**: `POST /numbers/whatsapp/bring-your-own/verify` (sms/voice OTP) → `…/confirm` with the code → `POST /channels/whatsapp` with `number_onboarding_mode: "bring_your_own"`, `waba_id`, `phone_number_id`, `access_token`. See `whatsapp/channels/byon` + `whatsapp/api/numbers`.
- **Track delivery**: store the returned `id`/`meta_message_id`, then correlate `message.status` webhooks by the `wamid` in `statuses[].id`.

## When to search docs

- "Send a template message" → `whatsapp/messaging/templates`, `whatsapp/api/send-message`
- "Inbound webhook / verify signature" → `whatsapp/webhooks`, `whatsapp/webhooks/events`
- "Port my number to Vobiz" → `whatsapp/channels/byon`, `whatsapp/api/numbers`
- "Connect a channel via API / embedded signup" → `whatsapp/api/channels`
- "Build a chatbot flow" → `whatsapp/messaging`
- "Buy a WhatsApp number" → `whatsapp/api/numbers`
- "Pricing / per-message fees" → `whatsapp/getting-started/introduction`
