---
name: vobiz-voice-xml
description: Build declarative call flows with VobizXML - Vobiz's TwiML-equivalent for play, speak, gather, dial, record, stream, conference, redirect, and hangup verbs.
license: MIT
compatibility: VobizXML 1.0, served from any HTTPS endpoint as application/xml.
metadata:
  author: vobiz
  version: "1.0"
---

# Vobiz Voice XML skill

Use this when the user wants to **script a call flow declaratively** - IVR menus, voicemail, queues, conferences, agent transfers. Anything beyond just "make a call and stream audio" usually involves VobizXML.

## How Vobiz fetches XML

1. Call is created via `POST /Call/` with `answer_url` (or an inbound call hits a number attached to an Application with an `answer_url`).
2. When the call is answered, Vobiz requests `answer_url` with the call context and `Event=StartApp`.
3. Your server returns `<Response>...verbs...</Response>` as `application/xml`.
4. Verbs execute top-to-bottom.

## Core verbs

| Verb | Purpose |
|---|---|
| `<Speak>` | TTS - speak text to the caller |
| `<Play>` | Play an audio file (HTTPS URL) |
| `<Gather>` | Collect DTMF or speech input |
| `<Dial>` | Bridge to another party - nest `<Number>` (PSTN) or `<User>` (SIP URI / registered WebRTC or SIP user) |
| `<Record>` | Record the call (optionally stream-while-recording) |
| `<Stream>` | Open a WebSocket for bidirectional audio (AI voice agents) |
| `<Conference>` | Add caller to a conference room |
| `<Redirect>` | Fetch new XML from a different URL |
| `<Wait>` | Pause for N seconds |
| `<Hangup>` | End the call |
| `<PreAnswer>` | Run `Speak`/`Play`/`Wait` before answering (early media, no billsec) |
| `<DTMF>` | Send tones onto the live call |

## Request payload - what your `answer_url` receives

Standard `application/x-www-form-urlencoded` POST (GET sends them as query params). Common parameters: `CallUUID`, `From`, `To`, `Direction`, `CallStatus`, `Event`, `CallerName`, `ForwardedFrom`, plus `ALegUUID`/`ALegRequestUUID` for outbound calls and any custom `X-VH-` SIP headers. `HangupCause`, `Duration`, `BillDuration` are present only after the call ends. See `xml/request` for the full list.

## Standard callback params by verb

Action/callback URLs receive the standard call params **plus** verb-specific ones:

- `<Gather action>`: `InputType` (dtmf|speech), `Digits`, `Speech`, `SpeechConfidenceScore`, `BilledAmount`. Empty `Digits`/`Speech` on timeout.
- `<Dial action>` (one final request, `Event=DialAction`): `DialStatus` (completed|busy|failed|cancel|timeout|no-answer), `DialRingStatus`, `DialHangupCause`, `DialALegUUID`, `DialBLegUUID` (empty if unanswered).
- `<Dial callbackUrl>` (real-time): `Event=DialAnswer|DialConnected|DialDigitsMatch|DialHangup` with `DialAction=answer|connected|digits|hangup`, plus `DialBLegUUID`, `DialBLegFrom`, `DialBLegTo`, `DialBLegPosition`; hangup events add `DialBLegDuration`, `DialBLegBillDuration`, `DialBLegHangupCause`.
- `<Record action>` (initial event, `Event=Record`): `RecordingID` only - do not treat it as "file ready".
- `<Record callbackUrl>` (`Event=RecordStop`): `RecordingID`, `RecordFile`/`RecordUrl`, `RecordingDuration`, `RecordingDurationMs`, `RecordingStartMs`, `RecordingEndMs`, `RecordingEndReason` (RecordingTimeout|maxLength|FinishedOnKey|HungUp).
- `<Conference callbackUrl>`: `ConferenceAction` (enter|exit) with `Event=ConferenceEnter|ConferenceExit`, `ConferenceUUID`, `ConferenceName`, `ConferenceMemberID`, `ConferenceFirstMember` (on enter), `ConferenceLastMember` (on exit). Derive room start/end from first enter and last exit.
- `<Stream statusCallbackUrl>`: `Event=StartStream|PlayedStream|ClearedAudio|DegradedStream|DroppedStream|StopStream` with `StreamID`, `CallUUID`, `Error`.
- `<Redirect>`: standard params + `Event=Redirect`.

## Per-verb quick recipes

- **IVR menu**: `<Gather numDigits="1" executionTimeout="10" action="...">` wrapping a `<Speak>`; put fallback `<Speak>`+`<Hangup/>` after it.
- **Variable-length input**: `<Gather finishOnKey="#" numDigits="N" executionTimeout="20">`.
- **Speech intent**: `<Gather inputType="speech" hints="..." language="en-US" speechEndTimeout="auto">`; threshold `SpeechConfidenceScore`. `inputType="dtmf speech"` accepts either.
- **Blind transfer**: `<Dial callerId="+1..." timeout="30" action="..."><Number>...</Number></Dial>` then fallback after.
- **Confirmation prompt to the answering agent**: `<Dial confirmSound="..." confirmKey="1">` - `confirmSound` returns `Speak`/`Play`/`Wait` XML; check `xml/dial/confirm-to-answer-call` before relying on `confirmKey` as an acceptance gate.
- **Voicemail**: `<Speak>` prompt + `<Record maxLength="120" timeout="10" finishOnKey="#" playBeep="true" callbackUrl="..."/>`; read the file URL from the `RecordStop` callback.
- **Record whole call**: self-closing `<Record recordSession="true" redirect="false" playBeep="false" maxLength="3600" callbackUrl="..."/>` as the first element, before `<Dial>`/`<Stream>`/`<Conference>`.
- **Hold music**: `<Play loop="0">...</Play>` or `<Speak loop="0">`.
- **AMD voicemail drop**: `<Wait beep="true" length="15"/>` then `<Speak>`.
- **Reject without billing**: `<Hangup reason="busy"/>` (or `rejected`) as the first element.
- **Branch/loop**: `<Redirect>https://.../next</Redirect>` (last runnable element).
- **Send tones to a remote IVR**: `<DTMF async="false">1WW2#</DTMF>`.
- **AI voice agent**: `<Stream bidirectional="true" keepCallAlive="true" contentType="audio/x-l16;rate=16000">wss://...</Stream>`.
- **Conference**: `<Conference stayAlone="true" callbackUrl="...">RoomName</Conference>`; set `startConferenceOnEnter="false"` + `waitSound` for participants who wait for a moderator.

## Pitfalls

- Response must be `application/xml` (or `text/xml`), not `text/html`/`text/plain`, served over HTTPS, and under 100 KB. Return within 1-2 s or callers hear dead air.
- **`<Gather>` uses `executionTimeout` (5-60 s, default 15), NOT `timeout`.** The timer starts after nested `Play`/`Speak` finish. `timeout` on `<Dial>` is the ring timeout (effective default 120 s when omitted - set it explicitly); `timeout` on `<Record>` is the silence timeout. `<Dial timeLimit>` (default 14400) caps the connected bridge only - don't confuse the two.
- Verbs execute strictly top-to-bottom. `<Redirect>` ends the current document. `Gather` and `Dial` default to `redirect="true"`, so the XML returned by `action` replaces the flow; elements after them run only when no handoff happens (no input, no bridge). `Record` defaults to `redirect="false"`: its `action` handler must return an empty `<Response></Response>`. Set `redirect="false"` on `Dial`/`Gather` when you only want the result delivered and the current flow to continue.
- All URLs in `<Play>`, `<Record>`, `<Gather>`, `<Dial>` (action/callbackUrl/confirmSound/dialMusic) must be fully qualified HTTPS. `dialMusic` and `confirmSound` must return XML (`Play`/`Speak`/`Wait`), not an audio file.
- **`<Dial callerId>` must be a Vobiz number your account owns or is authorized for** (E.164). If omitted on an inbound call, Vobiz derives it from the A-leg (the caller's number) and B-leg creation fails with no Dial callbacks - execution falls through to the next element.
- DTMF: `finishOnKey` (default `#`) is excluded from `Digits`; set `finishOnKey=""`/`none` to rely on `numDigits`/timeout. On no input, Vobiz still posts with empty `Digits`/`Speech` - check for empty.
- `Dial`: always branch on every `DialStatus` (busy/no-answer/timeout/failed/cancel/completed). `DialBLegUUID` is empty when nobody answers. Sibling `<Number>` elements dial simultaneously; losing legs report `DialBLegHangupCause=LOSE_RACE`.
- `Dial sipHeaders`/`User sipHeaders`: each key must **end** with `X-VH` (`RefX-VH=abc`); it arrives as `X-VH-Ref`. A key without the suffix drops the whole header set silently.
- `Record`: `timeout` is *silence* timeout (default 60 s, not total length); `maxLength` is the hard cap (default 60 s). `finishOnKey` defaults to every key (`1234567890*#`) - narrow it to `#` for voicemails. Set `maxLength` high enough for voicemails.
- `Conference`: room names are case-sensitive and the room is created on first join. `stayAlone` defaults to `false`, so a lone first arrival is disconnected - set `stayAlone="true"` on legs that may arrive first. `endConferenceOnExit="true"` removes every remaining member. Record conferences with a per-leg `<Record recordSession="true"/>` placed before `<Conference>`.
- `Stream`: without `keepCallAlive="true"` (requires `bidirectional="true"`) the call reaches end-of-XML and hangs up right after the stream starts. `contentType` sets the **inbound** format only (`audio/x-l16;rate=8000|16000` or `audio/x-mulaw;rate=8000`); 24 kHz L16 is for outbound `playAudio` only. Vobiz sends no inbound `stop` event - treat the WebSocket `close` plus the `hangup_url` `Event=Hangup` webhook as end-of-call.
- Escape `&`, `<`, `>` in `<Speak>`/`<Play>` text or the document fails to parse.
- `<Speak>` defaults to `voice="WOMAN"`, `language="en-US"`; not every language has both MAN and WOMAN voices - Vobiz falls back to the available one.
- `<PreAnswer>` early media depends on the carrier and number type, and WebRTC (Browser SDK) calls have none - the flow must work whether or not it is honored. Only `Speak`/`Play`/`Wait` nest inside it.
- `loop="0"` means infinite (`Play`, `Speak`) - it stops only on hangup or a parallel event, never on its own.
- `<Hangup schedule="N">` needs a following element (`Speak`/`Play`/`Wait`) or the call ends immediately.

## When to search docs

- "How do I build an IVR?" → `xml/gather`, `examples/vobiz-ivr-xml-python`
- "Voicemail" → `xml/record/record-a-voicemail`, `examples/vobiz-voicemail-xml-python`
- "Conference call" → `xml/conference`, `xml/conference/attributes`, `solutions/conference-calling`
- "AI voice agent" → `xml/stream`, `xml/stream/audio-formats`, `integrations/pipecat`
- "DTMF input" → `xml/dtmf`, `xml/gather/detecting-speech-inputs`
- "Transfer a live call" → `xml/dial/transferred-calls`, `call/transfer-call`
