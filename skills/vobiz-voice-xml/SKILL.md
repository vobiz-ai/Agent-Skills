---
name: vobiz-voice-xml
description: Build declarative call flows with VobizXML - Vobiz's TwiML-equivalent for play, speak, gather, dial, record, stream, conference, redirect, and hangup verbs.
---

# Vobiz Voice XML skill

Use this when the user wants to **script a call flow declaratively** - IVR menus, voicemail, queues, conferences, agent transfers. Anything beyond just "make a call and stream audio" usually involves VobizXML.

## How Vobiz fetches XML

1. Call is created via `POST /Call/` with `answer_url`.
2. When the callee answers, Vobiz POSTs to `answer_url` with call context.
3. Your server returns `<Response>...verbs...</Response>` as `application/xml`.
4. Verbs execute top-to-bottom.

## Core verbs

| Verb | Purpose |
|---|---|
| `<Speak>` | TTS - speak text to the caller |
| `<Play>` | Play an audio file (HTTPS URL) |
| `<Gather>` | Collect DTMF or speech input |
| `<Dial>` | Bridge to another party (PSTN, SIP URI, app, conference) |
| `<Record>` | Record the call (optionally stream-while-recording) |
| `<Stream>` | Open a WebSocket for bidirectional audio (AI voice agents) |
| `<Conference>` | Add caller to a conference room |
| `<Redirect>` | Fetch new XML from a different URL |
| `<Wait>` | Pause for N seconds |
| `<Hangup>` | End the call |
| `<Preanswer>` | Run verbs before answering (no billsec) |

## Request payload - what your `answer_url` receives

Standard `application/x-www-form-urlencoded` POST (GET sends them as query params). Common parameters: `CallUUID`, `From`, `To`, `Direction`, `CallStatus`, `HangupCause`, `Duration`, `BillDuration`, `ForwardedFrom`, plus `ALegUUID`/`ALegRequestUUID` for outbound calls and any custom `X-VH-` SIP headers. See `xml/request` for the full list.

## Standard callback params by verb

Action/callback URLs receive the standard call params **plus** verb-specific ones:

- `<Gather action>`: `InputType` (dtmf|speech), `Digits`, `Speech`, `SpeechConfidenceScore`, `BilledAmount`. Empty `Digits`/`Speech` on timeout.
- `<Dial action>`: `DialStatus` (completed|busy|failed|cancel|timeout|no-answer), `DialRingStatus`, `DialHangupCause`, `DialALegUUID`, `DialBLegUUID` (empty if unanswered).
- `<Dial callbackUrl>`: live `DialAction` (answer|connected|hangup|digits) events.
- `<Record action>`: `RecordUrl`, `RecordingID`, `RecordingDuration`(Ms), `RecordingEndReason` (RecordingTimeout|maxLength|FinishedOnKey|HungUp).
- `<Conference callbackUrl>`: `ConferenceAction` (enter|exit|start|end), `ConferenceUUID`, `ConferenceName`, `ConferenceCurrentSize`, `RecordingUrl` (on end).
- `<Redirect>`: standard params + `Event=Redirect`.

## Per-verb quick recipes

- **IVR menu**: `<Gather numDigits="1" executionTimeout="10" action="...">` wrapping a `<Speak>`; put fallback `<Speak>`+`<Hangup/>` after it.
- **Variable-length input**: `<Gather finishOnKey="#" numDigits="N" executionTimeout="20">`.
- **Speech intent**: `<Gather inputType="speech" hints="..." language="en-US">`; threshold `SpeechConfidenceScore`.
- **Blind transfer**: `<Dial timeout="30" action="..."><Number>...</Number></Dial>` then fallback after.
- **Screen the agent**: `<Dial confirmSound="..." confirmKey="1">`.
- **Voicemail**: `<Speak>` prompt + `<Record maxLength="120" finishOnKey="#" playBeep="true" action="...">`.
- **Record whole call**: `<Record recordSession="true" action="..." callbackUrl="...">` first.
- **Hold music**: `<Play loop="0">...</Play>` or `<Speak loop="0">`.
- **AMD voicemail drop**: `<Wait beep="true" length="15"/>` then `<Speak>`.
- **Reject without billing**: `<Hangup reason="busy"/>` as the first element.
- **Branch/loop**: `<Redirect>https://.../next</Redirect>` (last runnable element).
- **Send tones to a remote IVR**: `<DTMF async="false">1WW2#</DTMF>`.

## Pitfalls

- Response must be `application/xml` (or `text/xml`), not `text/html`/`text/plain`, served over HTTPS, and under 100 KB. Return within 1-2 s or callers hear dead air.
- **`<Gather>` uses `executionTimeout` (5-60 s, default 15), NOT `timeout`.** `timeout` belongs only to `<Dial>`/`<Number>` and is the ring timeout. `<Dial timeLimit>` caps total bridged duration - don't confuse the two.
- Verbs execute strictly top-to-bottom. `<Redirect>` and any `action`-URL handoff (`Dial`/`Gather`/`Record`) end the current document; elements after them are fallback-only.
- All URLs in `<Play>`, `<Record>`, `<Gather>`, `<Dial>` (action/callback/confirmSound/dialMusic) must be fully qualified HTTPS.
- DTMF: `finishOnKey` (default `#`) is excluded from `Digits`; set `finishOnKey=""`/`none` to rely on `numDigits`/timeout. On no input, Vobiz still posts with empty `Digits`/`Speech` - check for empty.
- `Dial`: always branch on every `DialStatus` (busy/no-answer/timeout/failed/cancel/completed). `DialBLegUUID` is empty when nobody answers.
- `Record`: `timeout` is *silence* timeout (not total length); `maxLength` is the hard cap. Set `maxLength` high enough for voicemails.
- Escape `&`, `<`, `>` in `<Speak>`/`<Play>` text or the document fails to parse.
- `<Speak>` defaults to `voice="WOMAN"`, `language="en-US"`; not every language has both MAN and WOMAN voices.
- `<PreAnswer>` early media is unsupported on WebRTC and some PSTN routes - the flow must work without it. Only `Speak`/`Play`/`Wait` nest inside it.
- `loop="0"` means infinite (`Play`, `Speak`) - it stops only on hangup or a parallel event, never on its own.

## When to search docs

- "How do I build an IVR?" → `xml/gather`, `examples/vobiz-ivr-xml-python`
- "Voicemail" → `xml/record/record-a-voicemail`, `examples/vobiz-voicemail-xml-python`
- "Conference call" → `xml/conference`, `solutions/conference-calling`
- "AI voice agent" → `xml/stream`, `integrations/pipecat`
- "DTMF input" → `xml/dtmf`, `xml/gather/detecting-speech-inputs`
