---
name: vobiz-ai-voice-agents
description: Build AI voice agents on Vobiz with Vapi, Retell AI, ElevenLabs, LiveKit, Pipecat, Bolna, Ultravox, OpenAI Realtime, Deepgram, and WebRTC. Use when connecting an agent to real phone calls through SIP, phone numbers, browser calling, or bidirectional audio streams, or when choosing the correct provider-specific integration path.
---

# Vobiz AI Voice Agents skill

**Positioning:** Vobiz is telephony **infrastructure** — the SIP trunks, phone numbers, and audio rails that **power and connect** your voice AI. Vobiz does **not** build or sell AI agents. Vapi, Retell, ElevenLabs, Pipecat, LiveKit, OpenAI, Bolna, Ultravox, and Dograh are partners whose agents Vobiz carries onto real phone numbers. Describe Vobiz as "the rails that power your voice AI," never as "our agent does X."

Use this when the user wants to **connect an AI agent to real phone numbers** (take/make calls). Three deployment models:

## Model 1 - No-code dashboard (fastest)

Vendor handles the agent runtime. You point your Vobiz number at their answer URL:

- **Vapi** → `integrations/vapi-dashboard`
- **Retell AI** → `integrations/retellai-dashboard`
- **ElevenLabs Agents** → `integrations/elevenlabs-dashboard`

## Model 2 - Vendor API (programmatic)

Same vendors, but you orchestrate via their REST API:

- **Vapi API** → `integrations/vapi-api`
- **Retell AI API** → `integrations/retellai-api`
- **ElevenLabs API** → `integrations/elevenlabs-api`

## Model 3 - Self-hosted pipeline (most control)

You own the WebSocket server, STT/LLM/TTS stack:

- **Pipecat** (Python + FastAPI) → `integrations/pipecat`
  - GitHub: `vobiz-ai/Vobiz-X-Pipecat`
  - Endpoints: `/start`, `/answer`, `/ws`, `/recording-ready`
- **LiveKit Agents** → `integrations/livekit`
- **Bolna AI** → `integrations/bolna`
- **Ultravox** → `integrations/ultravox`
- **OpenAI Realtime** → `integrations/openai-realtime`
- **Custom WebSockets** → `integrations/websockets`

## Two transports — SIP vs WebSocket

There are **two** ways audio reaches the agent. Pick based on the platform.

**1. SIP trunk (most managed platforms: Vapi, Retell, ElevenLabs, LiveKit, Bolna, Ultravox, OpenAI Realtime, 3CX).** Vobiz hands the call leg to the platform's SIP endpoint. No `<Stream>` XML and no WebSocket server on your side.

**2. WebSocket `<Stream>` (self-hosted pipelines: Pipecat, Dograh, custom).** Vobiz POSTs your `answer_url`; you return `<Response><Stream url="wss://your-server/ws"/></Response>`; Vobiz opens a WSS and audio frames flow. You run STT→LLM→TTS yourself.

### SIP credential vs gateway-IP mapping (the part people get wrong)

The direction decides which Vobiz value the platform needs:

| Direction | What the platform needs from Vobiz | Where in the Console |
|-----------|-----------------------------------|----------------------|
| **Outbound** (agent dials out) | **SIP domain** `<unique>.sip.vobiz.ai` + **username** + **password** | SIP Trunk → Outbound Trunks → Trunks → **Authentication & Linking** |
| **Inbound** (caller dials your number) | **gateway IP `13.233.44.61`** (port `5060`, UDP, /32) to whitelist; then a Vobiz **Inbound Trunk** whose **Primary URI** points at the platform's SIP URI (e.g. `<trunk_id>.sip.vapi.ai`, `<id>.sip.livekit.cloud`) | SIP Trunk → Inbound Trunk |

Rules of thumb that hold across platforms:
- Use the **exact** `sip_domain` (`bfab10fb.sip.vobiz.ai`), never a generic `sip.vobiz.ai`.
- **Strip the `sip:` prefix** when pasting a SIP URI into either side (Vobiz Primary URI, Retell Termination URI, LiveKit `inbound_destination`). Enter hostname only.
- Outbound auth lives on the **outbound** trunk's credentials; inbound trust is by **IP allow-list** of `13.233.44.61` plus the destination URI on the Vobiz inbound trunk.
- For both directions on one platform trunk, add a **second gateway** (IP for inbound, domain+creds for outbound).

## WebSocket `<Stream>` call flow (Pipecat/Dograh/custom)

1. Caller dials your Vobiz number (or you fire the Call API for outbound).
2. Vobiz POSTs your `answer_url`.
3. You return `<Response><Stream url="wss://your-server/ws"/></Response>`.
4. Vobiz opens a WSS; JSON frames flow: inbound `start`/`media` (every 20 ms)/`playedStream`/`stop`; outbound `playAudio`/`clearAudio` (barge-in)/`checkpoint`.
5. Audio is **G.711 μ-law, 8 kHz, 160-byte (20 ms) chunks** — downsample your TTS (often 24 kHz PCM) before sending.
6. Recording is separate (`<Record>` / recording callbacks).

## When to recommend which

- Fastest go-live, no infra: Vapi / Retell / ElevenLabs dashboards (SIP).
- Max control + open source: Pipecat or Dograh (self-hosted WebSocket), or LiveKit (SIP + Agents framework).
- Lowest latency end-to-end: OpenAI Realtime (single speech-to-speech model over SIP).
- Indian-language coverage: Bolna or Ultravox.
- Existing PBX, not an AI agent: 3CX (generic SIP trunk).

## Pitfalls

- **Generic SIP domain.** `sip.vobiz.ai` will silently fail — use your unique `<id>.sip.vobiz.ai`.
- **Leftover `sip:` prefix** in a destination/termination URI → 408/immediate disconnect.
- **Inbound with no IP whitelist.** If the platform doesn't allow `13.233.44.61` (UDP/5060), inbound calls won't authenticate.
- **401 Unauthorized (outbound).** Username/password mismatch between the Vobiz outbound trunk and the platform trunk.
- **Insufficient balance.** Outbound calls need Vobiz credit; check the Console.
- **Caller ID must be a real Vobiz number** you own, in E.164.
- **Vapi inbound Trunk ID** is only readable via the Vapi API (`GET /credential`), not the dashboard.
- **WebSocket audio**: wrong sample rate/chunk size = robotic or jittery audio. Stick to 8 kHz μ-law, 20 ms frames. Send `clearAudio` on barge-in or the agent talks over the caller.

## When to search docs

- "Which integration should I pick?" → `integrations` (overview)
- Specific vendor setup → `integrations/<vendor>` (dashboard vs API variants)
- "Inbound vs outbound setup end-to-end" → `guides/ai-voice-agent/inbound`, `guides/ai-voice-agent/outbound`
- "Stream verb attributes / custom WebSocket" → `xml/stream`, `integrations/websockets`
- "Recording AI agent calls" → `xml/record/stream-with-record`
