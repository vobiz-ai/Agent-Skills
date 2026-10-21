---
name: vobiz-audio-streams
description: Stream live call audio bidirectionally over WebSocket for AI voice agents, real-time transcription, and custom audio pipelines.
license: MIT
compatibility: Vobiz REST API v1, WebSocket transport, L16 / mu-law.
metadata:
  author: vobiz
  version: "1.1"
---

# Vobiz Audio Streams skill

Use this for **AI voice agents, real-time transcription, custom STT/TTS pipelines, or any workflow that needs raw audio frames** rather than just recordings.

## How it works

1. During a call, your VobizXML answer handler returns a `<Stream bidirectional="true" keepCallAlive="true">wss://your-server/ws</Stream>` element. The WebSocket URL is the element's **text content**, not a `url=""` attribute.
2. Vobiz opens a WebSocket to your server and sends a single `start` event with `callId`, `streamId`, `tracks`, and `mediaFormat` nested under `start`.
3. Audio frames flow Vobiz → your server as `media` events (~50 per second per track, 20 ms each).
4. For bidirectional streams, your server sends `playAudio` events back, plus `checkpoint`, `clearAudio`, and `stop` control messages.
5. When the call ends, the WebSocket **closes** - there is no inbound `stop` event.

## Two ways to start a stream

| Approach | When to use |
|---|---|
| **`<Stream>` XML element** | The call is being set up by your answer handler. Best for AI agents that own the call from the first ring. This is the documented path for full-duplex voice agents. |
| **REST `POST .../Stream/`** | The call is already live and you want to attach (or fork) audio mid-call - e.g. start transcription after a transfer, or stream a leg that is in a `<Conference>`. Returns a `stream_id` you can later list/get/stop. |

REST streams are forks: `bidirectional` over REST still requires `audio_track=inbound`, and **on the REST route `bidirectional` must be the JSON string `"true"`** - a JSON boolean `true` is accepted but silently produces a one-way stream (`audio-streams/start-audio-stream`). Both paths share the same WebSocket event protocol.

## REST endpoints

Base: `https://api.vobiz.ai/api/v1`. Auth headers: `X-Auth-ID`, `X-Auth-Token`.

| Op | Method + Path | Notes |
|---|---|---|
| Start stream | `POST /Account/{auth_id}/Call/{call_uuid}/Stream/` | Body needs `service_url`; returns `stream_id`. The doc page shows `202 Accepted`, `openapi.yaml` says `200` - treat either as success. |
| List streams | `GET /Account/{auth_id}/Call/{call_uuid}/Stream/` | Active + stopped, `limit` (default 20) / `offset` (default 0). |
| Get stream | `GET /Account/{auth_id}/Call/{call_uuid}/Stream/{stream_id}/` | Full Stream object; `404` if unknown. |
| Stop one stream | `DELETE /Account/{auth_id}/Call/{call_uuid}/Stream/{stream_id}/` | `204`; `404` if unknown, already stopped, or on another call. Leaves other forks running. |
| Stop all streams | `DELETE /Account/{auth_id}/Call/{call_uuid}/Stream/` | `204`, idempotent; tears down every fork on the call. Documented on `audio-streams/stop-all-audio-streams` (not in `openapi.yaml`). |

Request fields on start: `service_url` (required), `audio_track` (`inbound`/`outbound`/`both`, default `both`), `bidirectional` (string `"true"`/`"false"`), `content_type`, `stream_timeout` (default 86400), `status_callback_url`, `status_callback_method`, `extra_headers`. `openapi.yaml` still carries a legacy `audio_format: [pcm, mulaw]` enum; `content_type` is the real field (`OPENAPI-GAPS.md` item 17).

## Audio format

Treat the two directions separately. They are configured independently and may differ.

- **Inbound (Vobiz → you)** - set with `contentType` (XML) / `content_type` (REST): `audio/x-l16;rate=8000` (default), `audio/x-l16;rate=16000`, or `audio/x-mulaw;rate=8000`. **24 kHz is not a valid inbound format** - `rate=24000` on `<Stream>` connects but never delivers usable media. Vobiz reports the selected inbound format in `start.mediaFormat`; decode every `media.payload` from that.
- **Outbound (you → Vobiz)** - set per `playAudio` event with `media.contentType` (`audio/x-l16` or `audio/x-mulaw`) and `media.sampleRate` (L16: `8000`, `16000`, or `24000`; μ-law: `8000`). The payload bytes must genuinely match the declared rate - changing the JSON number does not resample.
- L16 is signed 16-bit linear PCM, mono. Byte order is not pinned down consistently across the docs (RFC 2586 says big-endian; `integrations/pipecat` reports some accounts transport little-endian and ships a `VOBIZ_L16_ENDIAN` switch; `integrations/gemini-live` sends little-endian). If L16 frames arrive at the expected size but STT returns nothing, flip the byte order. μ-law avoids the question entirely.
- 20 ms frame = 160 bytes μ-law @ 8 kHz, 320 bytes L16 @ 8 kHz, 640 bytes L16 @ 16 kHz, 960 bytes L16 @ 24 kHz. Send `playAudio` in roughly 20-60 ms chunks (a recommendation, not a protocol rule).
- Frames are base64-encoded raw samples inside `media.payload` / `playAudio.media.payload`. Never send WAV/MP3 container headers.
- 24 kHz playback does not guarantee 24 kHz at the handset - the PSTN/SIP leg may use a lower-rate codec.

## WebSocket events

**Vobiz → your server**

- `start` - one handshake frame. IDs live inside the nested `start` object: `data.start.streamId`, `data.start.callId`, `data.start.accountId`, `data.start.tracks`, `data.start.mediaFormat`. Top-level `sequenceNumber` starts at 0 and increments per message.
- `media` - base64 audio frame; top-level `streamId`, then `media.track`, `media.timestamp`, `media.chunk`, `media.payload`. Individual `media` events do not repeat the format.
- `playedStream` - ack that audio queued before your `checkpoint` finished playing. Shape is just `event` + `name` (no `streamId`). Match on `name`.
- `clearedAudio` - ack that `clearAudio` flushed the playback queue: `{ event, streamId, sequenceNumber }`.

**Your server → Vobiz** (require `bidirectional="true"`; all carry the `streamId` from `start`)

- `playAudio` - queue a chunk: `{ event, streamId, media: { contentType, sampleRate, payload } }`.
- `checkpoint` - mark end of an utterance: `{ event, streamId, name }`; ack arrives as `playedStream`.
- `clearAudio` - barge-in flush: `{ event, streamId }`; ack arrives as `clearedAudio`.
- `stop` - end the stream from your side: `{ event, streamId }`. Vobiz then runs the next XML element, or hangs up with `HangupCauseCode=4010` ("End Of XML Instructions") if there is none.

## HTTP status callbacks (separate channel)

If you set `statusCallbackUrl`, Vobiz POSTs form-encoded lifecycle events - distinct from WebSocket events. Every callback carries `Event`, `StreamID`, `CallUUID`, `From`, `To`, `Timestamp`, and an `Error` field (empty on healthy events):

- `Event=StartStream` - WebSocket established (carries `ServiceURL`).
- `Event=PlayedStream` - checkpoint reached (carries `Name`); only fires if playback completed.
- `Event=ClearedAudio` - fires on every barge-in flush; a busy call produces many of these.
- `Event=DegradedStream` (`Error=connection degraded`) - media path deteriorating while the stream is still up.
- `Event=DroppedStream` (`Error=connection disconnected with the remote service`) - also what you get when your own application closes the socket.
- `Event=StopStream` - stream ended. **Only reliably fires for server-initiated stops** (your `stop` command or a REST `DELETE`) - not for caller-hangup or mid-call kill.

The authoritative "call is over" signal across all paths is the `Event=Hangup` POST to your call's `hangup_url`. Full payloads: `xml/stream#status-callback-events`.

## When to search docs

- "Start mid-call over REST" → `audio-streams/start-audio-stream`; Stream fields → `audio-streams/stream-object`
- "Stop one fork vs all forks" → `audio-streams/stop-audio-stream`, `audio-streams/stop-all-audio-streams`
- "`<Stream>` attributes / status callbacks / maxRetries / streamTimeout / extraHeaders" → `xml/stream`
- "Which sample rate in which direction" → `xml/stream/audio-formats`, `concepts/streaming-websockets`
- "Handshake, end-of-stream detection, reconnects" → `xml/stream/initiate`
- "Full event protocol both directions" → `xml/stream/stream-events`
- "Playing audio mid-stream" → `xml/stream/play-audio`
- "Clearing queued TTS on interruption / barge-in" → `xml/stream/clear-audio`
- "Checkpoint / playback acks" → `xml/stream/checkpoint-event`
- "Ending the stream from your side" → `xml/stream/stop-event`
- "Record and stream at the same time" → `xml/record/stream-with-record`
- "Pipe audio to a model or framework" → `integrations/websockets`, `integrations/pipecat`, `integrations/gemini-live`, `integrations/deepgram`, `integrations/dograh`, `integrations/rapida-ai`
- "Firewall rules for an IP-restricted WebSocket server" → https://www.vobiz.ai/docs/concepts/ip-whitelisting#websocket-streaming (inbound TCP 443 from the media fleet; the RTP rule does not cover it)

## Pitfalls

- **No inbound `stop` event.** When the call ends, the WebSocket simply **closes** after the last `media` frame. Do not wait for `{ "event": "stop" }` from Vobiz - you will wait forever. Treat WS `close` as your end-of-stream signal and flush buffers there.
- **`StopStream` callback is conditional.** It fires reliably only when *you* stop the stream (WebSocket `stop` or REST `DELETE`). It is *not* observed for caller-hangup or mid-call kills. Use the `hangup_url` `Event=Hangup` webhook as the authoritative end-of-call signal.
- **IDs are nested.** Read `data.start.streamId` / `data.start.callId`, not `data.streamId`, on the `start` event.
- **`playedStream` is conditional.** No ack arrives if a `clearAudio` or disconnect interrupted the queued audio. Always have a timeout fallback - never block conversation logic on a checkpoint ack. Use unique checkpoint names per utterance.
- **Gate new playback on `clearedAudio`.** `playAudio` sent immediately after `clearAudio` can race the flush and be partially dropped. Wait for the ack (with a short timeout), then enqueue the replacement.
- **`bidirectional` excludes `both`/`outbound`.** With `bidirectional="true"` you must use `audioTrack="inbound"` (or omit it). `audioTrack="both"` + `bidirectional="true"` either hangs up with "End Of XML Instructions" or sends `start` and then never sends media. To capture both legs, use two one-way streams.
- **`keepCallAlive` needs `bidirectional="true"`.** Without `keepCallAlive`, a `<Stream>` with no following elements ends the XML and hangs up instantly. Use it instead of a `<Pause>`.
- **Inbound and outbound formats are independent.** `start.mediaFormat` tells you how to *decode* `media`. It does not set the `playAudio` format - declare that per event, and make the bytes match the declared `contentType`/`sampleRate`. Mismatched rate, stereo, or a WAV header = slow, fast, garbled, or silent playback.
- **Never configure 24 kHz inbound.** `contentType="audio/x-l16;rate=24000"` is accepted by the XML parser but no usable media arrives. For a 24 kHz TTS engine use 16 kHz in, 24 kHz out on `playAudio`.
- **REST `bidirectional` is a string.** Send `"bidirectional": "true"`; a JSON boolean silently yields a one-way stream and your `playAudio` is discarded.
- **Never detect end-of-call from `media.payload`.** All-`0xFF` (base64 all-`/`) bytes are just silence and appear mid-call. Use the WS `close` event.
- **Chunk size matters for barge-in.** Send ~20-60 ms chunks so a `clearAudio` flush is responsive. Large chunks add latency that `clearAudio` cannot recover.
- **`wss://` in production.** `ws://` is for local testing only. The URL must be publicly reachable; if it is IP-restricted, allow inbound TCP 443 from the media fleet per https://www.vobiz.ai/docs/concepts/ip-whitelisting#websocket-streaming.
- **`stop` is terminal.** After you send `stop`, the WS closes immediately - any queued `playAudio`/`checkpoint`/`clearAudio` sends are dropped. Send a farewell + `checkpoint`, wait for `playedStream`, then `stop`.
- **Reconnects.** `maxRetries` (XML, 0-10, default 0) controls how many times Vobiz retries a failed/dropped WebSocket. Each retry replays a fresh `start` with a **new `streamId`** and the same `callId` - key per-call state on `callId` and make your `start` handler idempotent. With the default `0`, a dropped socket is terminal.
- **`extraHeaders` are not socket credentials.** `integrations/deepgram` reports they surface only on `statusCallbackUrl` payloads (`X-VH-<key>`) and the `start` frame's `extra_headers` stays `"{}"`. Put a secret in the `wss://` URL path instead if you need to authenticate the socket.
- **Malformed answer XML is not an HTTP error.** Vobiz accepts the `200`, then drops the call; the CDR shows `hangup_cause_name: "Invalid Answer XML"`. A bare `&` in a URL with two query parameters is enough - XML-escape interpolated values.
- **`playAudio` bypasses the conference mixer.** It is injected into the leg the stream rides, so `deaf` cannot silence it and a stream can only address its own leg (`xml/stream/play-audio`).

## Complete bidirectional voice-agent recipe

A minimal full-duplex loop: greet on `start`, transcribe inbound `media`, barge-in with `clearAudio`, respond with `playAudio` + `checkpoint`, end with `stop`.

```javascript Node.js bidirectional agent skeleton
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

// Outbound playback format is independent of start.mediaFormat.
// Pick what your TTS actually emits: L16 8000/16000/24000 or mu-law 8000.
const OUT = { contentType: 'audio/x-l16', sampleRate: 8000 };
const FRAME_BYTES = OUT.sampleRate * 2 * 0.02;   // 20 ms of 16-bit mono L16

wss.on('connection', (ws) => {
  let streamId = null;
  let mediaFormat = null;   // { encoding, sampleRate } - how to DECODE inbound media
  let botSpeaking = false;
  let awaitingClear = false;
  let checkpointTimer = null;

  ws.on('message', async (raw) => {
    const data = JSON.parse(raw);

    switch (data.event) {
      case 'start':
        streamId = data.start.streamId;          // IDs are NESTED
        mediaFormat = data.start.mediaFormat;    // decode inbound media with this
        await speak(ws, await tts('Hi, how can I help?'), 'greeting');
        break;

      case 'media': {
        const pcm = Buffer.from(data.media.payload, 'base64'); // format per mediaFormat
        // Barge-in: caller talks while bot is talking (debounce with a real VAD).
        if (botSpeaking && isSpeech(pcm)) {
          awaitingClear = true;
          ws.send(JSON.stringify({ event: 'clearAudio', streamId }));
          botSpeaking = false;
        }
        feedTranscriber(pcm, mediaFormat); // your STT; on final transcript -> respond()
        break;
      }

      case 'clearedAudio':
        // Playback queue flushed; only now is it safe to enqueue the new response.
        awaitingClear = false;
        break;

      case 'playedStream':
        // Utterance finished. Clear any fallback timer.
        if (checkpointTimer) { clearTimeout(checkpointTimer); checkpointTimer = null; }
        botSpeaking = false;
        if (data.name === 'farewell') {
          ws.send(JSON.stringify({ event: 'stop', streamId }));
        }
        break;
    }
  });

  ws.on('close', () => {
    // Canonical end-of-stream. Flush recordings/transcripts here.
    teardown(streamId);
  });

  async function respond(text, name) {
    await speak(ws, await tts(text), name);
  }

  async function speak(ws, pcmBuffer, name) {
    botSpeaking = true;
    for (let i = 0; i < pcmBuffer.length; i += FRAME_BYTES) {
      ws.send(JSON.stringify({
        event: 'playAudio',
        streamId,                                   // required on every command
        media: {
          contentType: OUT.contentType,
          sampleRate: OUT.sampleRate,               // must match the real bytes
          payload: pcmBuffer.subarray(i, i + FRAME_BYTES).toString('base64'),
        },
      }));
    }
    ws.send(JSON.stringify({ event: 'checkpoint', streamId, name }));
    // Fallback: never block forever on a (conditional) playedStream ack.
    checkpointTimer = setTimeout(() => { botSpeaking = false; }, 15000);
  }
});
```

Pair it with this answer handler:

```xml VobizXML answer_url response
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Stream
      bidirectional="true"
      audioTrack="inbound"
      keepCallAlive="true"
      contentType="audio/x-l16;rate=8000"
      statusCallbackUrl="https://your-server.com/stream-status"
      maxRetries="3">
wss://your-server.com/ws
    </Stream>
</Response>
```
