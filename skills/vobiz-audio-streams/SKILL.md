---
name: vobiz-audio-streams
description: Stream live Vobiz call audio bidirectionally over WebSocket. Use when building AI voice agents, real-time transcription, speech analytics, custom STT/TTS pipelines, media playback, checkpoints, barge-in, stream event handling, or any workflow that needs raw audio frames instead of completed recordings.
---

# Vobiz Audio Streams skill

Use this for **AI voice agents, real-time transcription, custom STT/TTS pipelines, or any workflow that needs raw audio frames** rather than just recordings.

## How it works

1. During a call, your VobizXML answer handler returns a `<Stream bidirectional="true">wss://your-server/ws</Stream>` verb.
2. Vobiz opens a WebSocket to your server and sends a single `start` event with `callId`, `streamId`, and `mediaFormat`.
3. Audio frames flow Vobiz → your server as `media` events (~50 per second per track, 20 ms each).
4. For bidirectional streams, your server sends `playAudio` events back, plus `checkpoint`, `clearAudio`, and `stop` control messages.
5. When the call ends, the WebSocket **closes** - there is no inbound `stop` event.

## Two ways to start a stream

| Approach | When to use |
|---|---|
| **`<Stream>` XML verb** | The call is being set up by your answer handler. Best for AI agents that own the call from the first ring. Bidirectional playback requires this path. |
| **REST `POST .../Stream/`** | The call is already live and you want to attach (or fork) audio mid-call - e.g. start transcription after a transfer. Returns a `stream_id` you can later list/get/stop. |

REST streams are forks: `bidirectional=true` over REST still requires `audio_track=inbound`. For full duplex voice agents, prefer the XML verb.

## REST endpoints

Base: `https://api.vobiz.ai/api/v1`. Auth headers: `X-Auth-ID`, `X-Auth-Token`.

| Op | Method + Path | Notes |
|---|---|---|
| Start stream | `POST /Account/{auth_id}/Call/{call_uuid}/Stream/` | Body needs `service_url`; returns `stream_id` (202). |
| List streams | `GET /Account/{auth_id}/Call/{call_uuid}/Stream/` | Active + stopped, paginated (`limit`/`offset`). |
| Get stream | `GET /Account/{auth_id}/Call/{call_uuid}/Stream/{stream_id}/` | Full Stream object; 404 if unknown. |
| Stop one stream | `DELETE /Account/{auth_id}/Call/{call_uuid}/Stream/{stream_id}/` | 204; leaves other forks running. |
| Stop all streams | `DELETE /Account/{auth_id}/Call/{call_uuid}/Stream/` | 204; tears down every fork on the call. |

## Audio format

- `contentType` (XML) / `content_type` (REST): `audio/x-l16;rate=8000` (default), `audio/x-l16;rate=16000`, `audio/x-l16;rate=24000`, or `audio/x-mulaw;rate=8000`.
- L16 is 16-bit linear PCM, mono. **On the wire `media.payload` is network byte order (big-endian)** - swap to little-endian before writing a WAV. `playAudio` you send should be little-endian L16.
- µ-law is fixed at 8 kHz, mono. Recommended for carrier compatibility and lowest bandwidth.
- 20 ms frame = 160 bytes µ-law @ 8 kHz, 320 bytes L16 @ 8 kHz, 640 bytes L16 @ 16 kHz.
- Frames are base64-encoded inside `media.payload`. Never send WAV headers - raw samples only.
- The format Vobiz reports in `start.mediaFormat` is the format you must match in every `playAudio`.

## WebSocket events

**Vobiz → your server**

- `start` - one handshake frame. IDs live inside the nested `start` object: `data.start.streamId`, `data.start.callId`, `data.start.mediaFormat`.
- `media` - base64 audio frame; `media.track`, `media.timestamp`, `media.chunk`, `media.payload`.
- `playedStream` - ack that audio before your `checkpoint` finished playing. Shape is just `event` + `name` (no `streamId` on the WS event).
- `clearedAudio` - ack that `clearAudio` flushed the playback queue.

**Your server → Vobiz** (require `bidirectional="true"`)

- `playAudio` - queue a ~20 ms chunk: `{ event, media: { contentType, sampleRate, payload } }`.
- `checkpoint` - mark end of an utterance: `{ event, streamId, name }`; ack arrives as `playedStream`.
- `clearAudio` - barge-in flush: `{ event, streamId }`; ack arrives as `clearedAudio`.
- `stop` - end the stream from your side: `{ event, streamId }`. Vobiz then runs the next XML element, or hangs up.

## HTTP status callbacks (separate channel)

If you set `statusCallbackUrl`, Vobiz POSTs form-encoded lifecycle events - distinct from WebSocket events:

- `Event=StartStream` - WebSocket established (carries `ServiceURL`).
- `Event=PlayedStream` - checkpoint reached (carries `Name`); only fires if playback completed.
- `Event=StopStream` - stream ended. **Only reliably fires for server-initiated stops** - not for caller-hangup or mid-call kill.

The authoritative "call is over" signal across all paths is the `Event=Hangup` POST to your call's `hangup_url`.

## When to search docs

- "Start mid-call over REST" → `audio-streams/start-audio-stream`; Stream fields → `audio-streams/stream-object`
- "Stop one fork vs all forks" → `audio-streams/stop-audio-stream`, `audio-streams/stop-all-audio-streams`
- "`<Stream>` attributes / status callbacks / maxRetries" → `xml/stream`
- "Handshake, codecs, end-of-stream detection" → `xml/stream/initiate`
- "Full event protocol both directions" → `xml/stream/stream-events`
- "Playing audio mid-stream" → `xml/stream/play-audio`
- "Clearing queued TTS on interruption / barge-in" → `xml/stream/clear-audio`
- "Checkpoint / playback acks" → `xml/stream/checkpoint-event`
- "Ending the stream from your side" → `xml/stream/stop-event`
- "Pipe audio to Deepgram / OpenAI Realtime / Pipecat / LiveKit" → `integrations/openai-realtime`, `integrations/websockets`, `integrations/pipecat`, `integrations/livekit`

## Pitfalls

- **No inbound `stop` event.** When the call ends, the WebSocket simply **closes** after the last `media` frame. Do not wait for `{ "event": "stop" }` from Vobiz - you will wait forever. Treat WS `close` as your end-of-stream signal and flush buffers there.
- **`StopStream` callback is conditional.** It fires reliably only when *you* send an outbound `stop`. It is *not* observed for caller-hangup or mid-call kills. Use the `hangup_url` `Event=Hangup` webhook as the authoritative end-of-call signal.
- **IDs are nested.** Read `data.start.streamId` / `data.start.callId`, not `data.streamId`, on the `start` event.
- **`playedStream` is conditional.** No ack arrives if a `clearAudio` or disconnect interrupted the queued audio. Always have a timeout fallback - never block conversation logic on a checkpoint ack.
- **`bidirectional` excludes `both`/`outbound`.** With `bidirectional="true"` you must use `audioTrack="inbound"` (or omit it). `audioTrack="both"` + `bidirectional="true"` makes the call hang up with "End Of XML Instructions".
- **`keepCallAlive` needs `bidirectional="true"`.** Without `keepCallAlive`, a `<Stream>` with no following elements ends the XML and hangs up instantly. Use it instead of a `<Pause>`.
- **Endianness bites.** `media.payload` L16 is big-endian inbound; `playAudio` L16 must be little-endian. Mismatched endianness = static/garbled audio.
- **Format must match `start.mediaFormat`.** Sending `playAudio` at a different `contentType`/`sampleRate` than negotiated produces garbled playback.
- **Never detect end-of-call from `media.payload`.** All-`0xFF` (base64 all-`/`) bytes are just silence and appear mid-call. Use the WS `close` event.
- **Chunk size matters for barge-in.** Send ~20 ms chunks so a `clearAudio` flush is responsive. Large chunks add latency that `clearAudio` cannot recover.
- **`wss://` in production.** `ws://` is for local testing only; carriers and Vobiz require TLS in production. The URL must be publicly reachable.
- **`stop` is terminal.** After you send `stop`, the WS closes immediately - any queued `playAudio`/`checkpoint`/`clearAudio` sends are dropped. Send a farewell + `checkpoint`, wait for `playedStream`, then `stop`.
- **Reconnects.** `maxRetries` (XML, 0-10, default 0) controls how many times Vobiz retries a failed/dropped WebSocket. Make your `start` handler idempotent - a reconnect replays a fresh `start` with a new `streamId`.

## Complete bidirectional voice-agent recipe

A minimal full-duplex loop: greet on `start`, transcribe inbound `media`, barge-in with `clearAudio`, respond with `playAudio` + `checkpoint`, end with `stop`.

```javascript Node.js bidirectional agent skeleton
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  let streamId = null;
  let mediaFormat = null;   // { encoding, sampleRate } from start
  let botSpeaking = false;
  let checkpointTimer = null;

  ws.on('message', async (raw) => {
    const data = JSON.parse(raw);

    switch (data.event) {
      case 'start':
        streamId = data.start.streamId;          // IDs are NESTED
        mediaFormat = data.start.mediaFormat;    // match this in playAudio
        await speak(ws, await tts('Hi, how can I help?'), 'greeting');
        break;

      case 'media': {
        const pcm = Buffer.from(data.media.payload, 'base64'); // big-endian L16
        // Barge-in: caller talks while bot is talking.
        if (botSpeaking && isSpeech(pcm)) {
          ws.send(JSON.stringify({ event: 'clearAudio', streamId }));
          botSpeaking = false;
        }
        feedTranscriber(pcm); // your STT; on final transcript -> respond()
        break;
      }

      case 'clearedAudio':
        // Playback queue flushed; safe to enqueue the new response.
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
    const frame = 320; // 20 ms of L16 @ 8 kHz; adjust to mediaFormat.sampleRate
    for (let i = 0; i < pcmBuffer.length; i += frame) {
      ws.send(JSON.stringify({
        event: 'playAudio',
        media: {
          contentType: mediaFormat.encoding,   // e.g. "audio/x-l16"
          sampleRate: mediaFormat.sampleRate,  // e.g. 8000
          payload: pcmBuffer.subarray(i, i + frame).toString('base64'), // little-endian
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
