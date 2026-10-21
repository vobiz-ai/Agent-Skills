---
name: vobiz-ai-voice-agents
description: Build AI voice agents on Vobiz with native integrations for Vapi, Retell AI, ElevenLabs, LiveKit, Pipecat, Bolna, Ultravox, OpenAI Realtime, Gemini Live, Deepgram, Sarvam, Dograh, Rapida AI, Smallest AI, Agora, xAI, 3CX, and WebRTC.
license: MIT
compatibility: Vobiz REST API + VobizXML <Stream> verb + WebSocket transport.
metadata:
  author: vobiz
  version: "1.0"
---

# Vobiz AI Voice Agents skill

**Positioning:** Vobiz is telephony **infrastructure** - the SIP trunks, phone numbers, and audio rails that **power and connect** your voice AI. Vobiz does **not** build or sell AI agents. Vapi, Retell, ElevenLabs, LiveKit, Pipecat, OpenAI, Google, Deepgram, Sarvam, Bolna, Ultravox, Dograh, Rapida AI, Smallest AI, Agora, and xAI are partners whose agents Vobiz carries onto real phone numbers. Describe Vobiz as "the rails that power your voice AI," never as "our agent does X."

Use this when the user wants to **connect an AI agent to real phone numbers** (take/make calls). Three deployment models:

## Model 1 - No-code dashboard (fastest)

Vendor handles the agent runtime. You pair a Vobiz trunk or Vobiz credentials with the vendor's console:

- **Vapi** (SIP, in + out) → `integrations/vapi-dashboard`
- **Retell AI** (SIP, in + out) → `integrations/retellai-dashboard`
- **ElevenLabs Agents** (SIP, in + out) → `integrations/elevenlabs-dashboard`
- **Smallest AI** (SIP, in + out, bring your own Vobiz number) → `integrations/smallest-ai`
- **Agora Conversational AI** (SIP, inbound agent + outbound campaigns) → `integrations/agora`
- **xAI Grok** (Direct SIP over TLS, inbound only) → `integrations/xai`
- **Bolna** (Vobiz Auth credentials entered as a provider; Bolna provisions numbers and handles in + out) → `integrations/bolna`
- **Sarvam Samvaad** ("Rent from Sarvam" is Vobiz embedded with hosted KYC; or connect your own Vobiz Auth ID / Auth Token) → `integrations/sarvam`

## Model 2 - Vendor API (programmatic)

Same idea, but you orchestrate via the vendor's REST API:

- **Vapi API** → `integrations/vapi-api`
- **Retell AI API** → `integrations/retellai-api`
- **ElevenLabs API** → `integrations/elevenlabs-api`
- **Ultravox** (outbound only; Vobiz SIP domain + credentials go in the Ultravox create-call request) → `integrations/ultravox`
- **OpenAI Realtime** (inbound only; SIP over TLS + SRTP to OpenAI, your server handles the `realtime.call.incoming` webhook and control WebSocket - no `<Stream>`) → `integrations/openai-realtime`

## Model 3 - Self-hosted pipeline (most control)

You own the WebSocket server and the STT/LLM/TTS stack, or run an open-source platform that does:

- **Pipecat** (Python + FastAPI) → `integrations/pipecat`
  - GitHub: `vobiz-ai/Vobiz-X-Pipecat`, serializer in the `pipecat-vobiz` package
  - Endpoints: `/start`, `/answer`, `/ws`, `/recording-ready`
- **Gemini Live bridge** (single speech-to-speech session; 16 kHz in, 24 kHz out) → `integrations/gemini-live` (GitHub: `vobiz-ai/Vobiz-Gemini-Live-Streaming`)
- **Deepgram Voice Agent bridge** (Flux STT/TTS, India region default) → `integrations/deepgram` (GitHub: `vobiz-ai/Vobiz-Deepgram-Voice-Agent`)
- **Dograh** (open-source, self-hosted platform; Docker) → `integrations/dograh`
- **Rapida AI** (open-source platform; Vobiz XML Application + WebSocket) → `integrations/rapida-ai`
- **LiveKit Agents** (SIP into LiveKit, agent runs as a room participant) → `integrations/livekit`
- **Custom WebSockets** → `integrations/websockets`

Not AI agents, but often asked about alongside them: **3CX** (business PBX on a generic SIP trunk) → `integrations/3cx`; **WebRTC Playground** (browser softphone) → `integrations/webrtc-application-setup`.

## Two transports - SIP vs WebSocket

There are **two** ways audio reaches the agent. Pick based on the platform (`concepts/sip-vs-websockets`).

**1. SIP trunk (Vapi, Retell, ElevenLabs, LiveKit, Ultravox, OpenAI Realtime, Smallest AI, Agora, xAI, 3CX).** Vobiz hands the call leg to the platform's SIP endpoint. No `<Stream>` XML and no WebSocket server on your side. Lower audio-transport latency (UDP/RTP), SIP REFER transfer, PBX compatibility.

**2. WebSocket `<Stream>` (Pipecat, Gemini Live, Deepgram, Dograh, Rapida AI, custom).** Vobiz POSTs your `answer_url`; you return `<Response><Stream bidirectional="true" keepCallAlive="true">wss://your-server/ws</Stream></Response>`; Vobiz opens a WSS and audio frames flow. You run (or your platform runs) STT→LLM→TTS. No platform markup on the Vobiz rate.

**Credential-based (Bolna, Sarvam, Rapida AI, Dograh).** You paste your Vobiz Auth ID / Auth Token into the platform; it places calls through the Vobiz REST API and attaches its own stream. No trunk to configure.

### SIP credential vs gateway-IP mapping (the part people get wrong)

The direction decides which Vobiz value the platform needs:

| Direction | What the platform needs from Vobiz | Where in the Console |
|-----------|-----------------------------------|----------------------|
| **Outbound** (agent dials out) | **SIP domain** `<unique>.sip.vobiz.ai` + **username** + **password** from a credential attached to the trunk | SIP Trunk → Outbound Trunks → Trunks → **Authentication & Linking** |
| **Inbound** (caller dials your number) | The platform must accept SIP from the **Vobiz SIP signaling IPs** (UDP/5060 unless the vendor says TLS) - current list at https://www.vobiz.ai/docs/concepts/ip-whitelisting#sip-signaling; then a Vobiz **Inbound Trunk** whose **Primary URI** (an Origination URI) points at the platform's SIP host, e.g. `<trunk_id>.sip.vapi.ai`, `<id>.sip.livekit.cloud`, `sip.retellai.com`, `sip.rtc.elevenlabs.io:5060`, `sbc-ap-south.viblinx.com` (Agora), `sip.api.openai.com:5061` | SIP Trunk → Inbound Trunks (Origination URIs at `console.vobiz.ai/app/sip/in/uri`, trunks at `/app/sip/in/trunks`), then **Link Numbers** |

Rules of thumb that hold across platforms:
- Use the **exact** `sip_domain` (`<id>.sip.vobiz.ai`), never a generic `sip.vobiz.ai`. Each trunk has its own domain.
- **Console fields and vendor dashboards take the hostname only** - strip `sip:` when pasting into a Vobiz Console URI field, a Retell Termination URI, an ElevenLabs Address, a LiveKit `inbound_destination`, or a 3CX Registrar. The Console URI field takes `domain` or `domain:port` and warns on a bare host - click **Save anyway**. The **Create Origination URI API** (`sip_uri`) is documented as expecting the `sip:` scheme; the two spellings are an open question in `OPENAPI-GAPS.md` - follow the vendor page you are on.
- Vobiz sends inbound SIP from **any** of its signaling IPs. Vapi needs one gateway per IP; LiveKit uses an **Allowed Addresses** list (an allow-all mask is fine while testing; tighten it in production); xAI uses **Allowed addresses**. Link to the whitelisting page rather than hard-coding addresses - they change.
- Outbound auth lives on the **outbound** trunk's credential; inbound trust is the platform's IP allow-list plus the Primary URI on the Vobiz inbound trunk.
- For both directions on one Vapi trunk, add a **second gateway** (IP gateways for inbound, domain + credentials gateway for outbound). Retell, Smallest AI, and Agora use **two separate Vobiz trunks** (outbound + inbound) on the same number.
- Transport differs by vendor: ElevenLabs `TCP`, Retell API `TCP` recommended, OpenAI Realtime and xAI `TLS`, Vapi / Smallest AI / Agora inbound `UDP`. Follow the vendor page.

### OpenAI Realtime specifics (`integrations/openai-realtime`)

- Origination URI `proj_xxxxxxxx@sip.api.openai.com:5061` with Transport **TLS**. The `:5061` port is mandatory - without it the CDR shows `hangup_disposition: send_refuse`.
- When a trunk has a primary Origination URI, the **URI's `transport`** decides signalling and the trunk's own `transport` is ignored. Set TLS on the URI.
- `secure: true` (SRTP) on the **trunk** is separate from `transport`. TLS without `secure` yields a connected, billed call with no audio and elevated `packet_loss`.
- `transport` is not settable on the Create Origination URI API - select it in the Console.

## WebSocket `<Stream>` call flow (Pipecat/Gemini/Deepgram/Dograh/custom)

1. Caller dials your Vobiz number (number attached to a Voice Application whose Answer URL is your server), or you fire the Call API for outbound with `answer_url`.
2. Vobiz POSTs your `answer_url`.
3. You return `<Response><Stream bidirectional="true" audioTrack="inbound" keepCallAlive="true" contentType="audio/x-mulaw;rate=8000">wss://your-server/ws</Stream></Response>`. The URL is the element's text content.
4. Vobiz opens a WSS; JSON frames flow: inbound `start` (IDs nested under `start`) / `media` (every 20 ms) / `playedStream` / `clearedAudio`; outbound `playAudio` / `checkpoint` / `clearAudio` (barge-in) / `stop`. There is **no inbound `stop`** - the socket closing is the end-of-stream signal.
5. Audio directions are independent. Inbound (`contentType`): L16 8 kHz (default), L16 16 kHz, or μ-law 8 kHz - never 24 kHz. Outbound (`playAudio.media`): L16 8/16/24 kHz or μ-law 8 kHz, ~20-60 ms chunks, raw mono, no WAV header. The Pipecat and Dograh references use μ-law 8 kHz both ways; the Gemini and Deepgram bridges use 16 kHz in / 24 kHz out so nothing is resampled.
6. Recording is separate (`<Record/>` as a self-closing sibling before `<Stream>`, or recording callbacks).

Full protocol, status callbacks, and pitfalls: the `vobiz-audio-streams` skill and `xml/stream`.

## When to recommend which

Per `integrations` (overview table) and `concepts/sip-vs-websockets`:

- Fastest go-live, no infra: Vapi or Retell dashboards (SIP); ElevenLabs for realistic TTS voices.
- Lowest-latency streaming: LiveKit (SIP into the Agents framework).
- Custom AI pipeline: Pipecat or custom WebSockets. Open-source agent platform: Rapida AI or Dograh.
- Single speech-to-speech model, no separate STT/TTS vendors: OpenAI Realtime (SIP, inbound only), Gemini Live, or Deepgram Voice Agent (WebSocket bridges, in + out).
- Indian-language agents with no telephony setup: Sarvam. India-first voice agents: Bolna. Deepgram's bridge defaults to the Hyderabad region with Indian-accented English voices and Hindi/Indic recognition.
- Multilingual speech model for outbound: Ultravox.
- Two-way agent on a number you already own: Smallest AI. Outbound campaigns from a voice agent: Agora. Grok voice agent on your own number: xAI.
- Above roughly 50,000 calls/month the WebSocket path (no platform markup) tends to pay off; below it, a managed platform usually wins on engineering time.
- Existing PBX, not an AI agent: 3CX (generic SIP trunk).

## Pitfalls

- **Generic SIP domain.** `sip.vobiz.ai` will silently fail - use your unique `<id>.sip.vobiz.ai`.
- **Leftover `sip:` prefix** in a Console URI field, Termination URI, Address, or `inbound_destination` → immediate disconnect or the call never routes.
- **Inbound with no IP allow-list.** If the platform does not accept SIP from the Vobiz signaling IPs (https://www.vobiz.ai/docs/concepts/ip-whitelisting#sip-signaling), inbound calls are rejected. Whitelist all of them - Vobiz can send from any.
- **RTP is a separate rule.** SIP signaling IPs are not the media IPs; if your network filters RTP, allow UDP 5000-65535 from the RTP media list (https://www.vobiz.ai/docs/concepts/ip-whitelisting#rtp-media). Do not add RTP ranges as SIP gateways.
- **401 Unauthorized (outbound).** Username/password mismatch between the Vobiz outbound trunk credential and the platform trunk, or the credential is not attached to the trunk.
- **Insufficient balance.** Outbound calls need Vobiz credit; check the Console.
- **Caller ID must be a real Vobiz number** you own, in E.164 (Dograh wants it without the `+`).
- **Vapi inbound Trunk ID** is only readable via the Vapi API (`GET /credential`), not the dashboard.
- **Inbound trunk with no Primary URI or no linked number** has nowhere to send the call - check **Linked Numbers** on the *inbound* trunk, not the outbound one.
- **Inbound and outbound agents are set separately** on Retell and Agora; an unassigned inbound agent accepts and drops the call.
- **Number with no Voice Application** (WebSocket path): Vobiz routes inbound by the application attached to the number - create the application, then attach the number.
- **WebSocket audio**: use `audioTrack="inbound"` (or omit it) with `bidirectional="true"` - combining it with `both` ends the call; `keepCallAlive="true"` is required or the call hits "End Of XML Instructions"; declared `sampleRate` must match the real bytes; send `clearAudio` on barge-in or the agent talks over the caller.
- **Malformed answer XML** is accepted with `200` and the call drops ~1 s later (`hangup_cause_name: "Invalid Answer XML"`) - XML-escape `&` in URLs.

## When to search docs

- "Which integration should I pick?" → `integrations` (overview)
- Specific vendor setup → `integrations/<vendor>` (dashboard vs API variants for Vapi, Retell, ElevenLabs)
- "Inbound vs outbound setup end-to-end" → `guides/ai-voice-agent/inbound`, `guides/ai-voice-agent/outbound`
- "SIP or WebSocket for my case" → `concepts/sip-vs-websockets`
- "Stream verb attributes / custom WebSocket" → `xml/stream`, `integrations/websockets`, `concepts/streaming-websockets`
- "Which IPs to whitelist" → https://www.vobiz.ai/docs/concepts/ip-whitelisting
- "Trunk fields (`transport`, `secure`, `primary_uri_uuid`, `inbound_destination`)" → `trunks/trunk-object`, `trunks/origination-uri/origination-uri-object`
- "Recording AI agent calls" → `xml/record/stream-with-record`
