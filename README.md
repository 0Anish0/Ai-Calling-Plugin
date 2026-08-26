# Vobiz + Sarvam AI — Hindi Voice Agent

A Hindi-first, bidirectional AI voice agent for Vobiz that pairs Sarvam Saaras v3 speech-to-text, OpenAI GPT-4o-mini, and Sarvam Bulbul v3 text-to-speech over a single WebSocket.

## Overview

Most voice-agent examples assume the caller speaks English. This one does not. It
answers an inbound call on a Vobiz number, opens a bidirectional media stream to
your own WebSocket, and runs a full speech loop in Hindi — or any of the other
languages Sarvam's Saaras v3 and Bulbul v3 models cover — with the caller
interrupting and being answered in real time.

The design is deliberately small and readable. Two files do everything: a FastAPI
app that returns the answer XML and hosts the WebSocket, and a per-call session
object that carries the audio buffer, the voice-activity detector, the
conversation history, and the three provider calls. There is no framework to
learn between the phone call and the model — you can read the whole pipeline in
about four hundred lines and change any stage of it.

Audio arrives from Vobiz as 8 kHz mu-law. A silence-based voice-activity detector
segments the caller's utterance, the segment is wrapped in a WAV container and
sent to Sarvam for transcription, the transcript is appended to a running
conversation and sent to GPT-4o-mini, and the reply is synthesised by Bulbul and
written back down the same socket as 20 ms mu-law frames. Every step is
inspectable from the logs.

At the end you have a working phone number a person can ring and hold a Hindi
conversation with, plus a codebase you can point at a different language, a
different voice, a different LLM, or your own business logic without unpicking
anything.

## What you can build with it

- **Hindi-language customer support line** — a first-tier agent that answers in
  Hindi, understands code-mixed Hinglish, and hands off to a human when needed.
- **Regional-language order and delivery status** — callers ask in Tamil,
  Marathi, or Bengali; the same code path serves all of them by changing one
  environment variable.
- **Appointment confirmation and reminders** — outbound or inbound flows where
  the caller confirms, reschedules, or cancels by speaking naturally.
- **Lead qualification for Indian markets** — collect intent and budget in the
  caller's own language before routing to a sales rep.
- **Internal helpline for field staff** — a spoken interface to an internal
  knowledge base for drivers, technicians, or agents who are not comfortable
  typing in English.
- **Bilingual switchboard** — set `AGENT_LANGUAGE` per deployment and run one
  number per language against identical code.

## How it works

Vobiz requests your answer URL when a call comes in. You return a `<Stream>`
element with `bidirectional="true"`, which tells Vobiz to open a WebSocket to the
URL you supply and to accept audio back on the same connection.
`keepCallAlive="true"` keeps the call parked on the stream, so the `<Hangup/>`
after it only runs once the stream disconnects.

Once the socket is open, Vobiz sends JSON events. `start` carries the stream and
call identifiers and triggers the greeting. `media` events carry base64 mu-law
audio, roughly one 20 ms frame at a time. Your side sends `playAudio` events with
audio to play, `clearAudio` to flush what is queued when the caller interrupts,
and `checkpoint` to be told when a queued run of audio has finished playing —
Vobiz answers that with `playedStream`.

```
Inbound call → POST /answer → <Stream bidirectional="true">
                                      │
                             WebSocket /ws
                                      │
                    ┌─────────────────▼─────────────────┐
                    │  CallSession (per call)            │
                    │                                    │
                    │  mu-law audio → VAD buffer         │
                    │       (800ms silence trigger)      │
                    │            ↓                       │
                    │  Sarvam Saaras v3  (STT)           │
                    │            ↓                       │
                    │  OpenAI GPT-4o-mini (LLM)          │
                    │            ↓                       │
                    │  Sarvam Bulbul v3  (TTS)           │
                    │            ↓                       │
                    │  playAudio → caller hears reply    │
                    └────────────────────────────────────┘
```

The voice-activity detector is the part that decides when a turn has ended. Each
inbound frame is converted to linear PCM and its RMS compared against
`SILENCE_THRESHOLD`. Frames above it count as speech and start filling the
buffer; once forty consecutive frames fall below it — 800 ms — and the buffer
holds at least eight speech frames, the buffer is handed to the pipeline as a
complete utterance and the detector resets.

## Architecture

| File | Responsibility |
|------|----------------|
| `server.py` | FastAPI application. Resolves the public base URL (explicit `PUBLIC_URL` or an ngrok tunnel), serves `/answer` with the Stream XML, accepts the WebSocket at `/ws` and feeds every frame to a `CallSession`, and logs the `/stream-status` and `/hangup` callbacks. Entry point. |
| `agent.py` | The `CallSession` class — per-call state machine holding VAD state, the audio buffer, playback state, and the conversation history. Contains the event router, the STT/LLM/TTS calls, the mu-law ⇄ WAV conversions, and the playback helpers. Also ships a standalone `websockets` server so the session can be hosted without FastAPI. |
| `.env.example` | Template for the runtime configuration. Copy to `.env` and fill in. |
| `LICENSE` | MIT licence text. |

`agent.py` has no dependency on `server.py`. The `_send` helper detects whether it
is holding a Starlette WebSocket (`send_text`) or a raw `websockets` connection
(`send`), so the same session class works under either host.

## Prerequisites

- **A Vobiz account** with a voice-enabled number and an Application whose answer
  URL points at `POST <your-public-url>/answer`. See the
  [Vobiz docs](https://docs.vobiz.ai).
- **A Sarvam AI subscription key** — used for both Saaras v3 (STT) and Bulbul v3
  (TTS). Sign up at [sarvam.ai](https://www.sarvam.ai/).
- **An OpenAI API key** with access to `gpt-4o-mini`.
- **Python 3.9–3.12.** The audio conversions use the standard-library `audioop`
  module, which was deprecated in 3.11 and **removed in Python 3.13**. On 3.13 or
  newer the import fails; use 3.12 or install a compatible `audioop` backport.
- **A public HTTPS URL.** Vobiz must reach your `/answer` endpoint and open a
  `wss://` connection to `/ws`. Either set `PUBLIC_URL` to a real deployment, or
  leave it empty and let the bundled ngrok tunnel provide one for development.

## Setup

1. **Clone the repository.**

   ```bash
   git clone https://github.com/vobiz-ai/Vobiz-Sarvam.git
   cd Vobiz-Sarvam
   ```

2. **Create a virtual environment on a supported Python.**

   ```bash
   python3.12 -m venv .venv
   source .venv/bin/activate
   ```

3. **Install the dependencies.** There is no `requirements.txt`; these are the
   packages the two files actually import.

   ```bash
   pip install fastapi uvicorn python-dotenv pyngrok httpx websockets openai
   ```

4. **Create the environment file.**

   ```bash
   cp .env.example .env
   ```

   Fill in `SARVAM_API_KEY` and `OPENAI_API_KEY`. Everything else has a working
   default.

   > **Note:** `server.py` and `agent.py` resolve `.env` relative to their own
   > location, so the file created above is picked up no matter which directory
   > you launch from.

5. **Give Vobiz a public URL.** For a real deployment, set `PUBLIC_URL` to your
   HTTPS origin with no trailing slash. For local development, leave it empty and
   set `NGROK_AUTH_TOKEN`; the server opens a tunnel on start and prints the URL.

6. **Point your Vobiz Application at the server.** Set the answer URL to
   `<base-url>/answer` with method `POST`, and the hangup URL to
   `<base-url>/hangup`. Assign the Application to your Vobiz number.

## Configuration

Every variable the code reads, with the default it falls back to.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `SARVAM_API_KEY` | Yes | *(empty)* | Sarvam AI subscription key. Sent as the `api-subscription-key` header on both the STT and TTS calls. |
| `OPENAI_API_KEY` | Yes | *(empty)* | OpenAI API key, passed to `AsyncOpenAI` at import time. |
| `AGENT_LANGUAGE` | No | `hi-IN` | BCP-47 language tag. Used as `language_code` for Saaras v3 and `target_language_code` for Bulbul v3. |
| `TTS_SPEAKER` | No | `anand` | Bulbul v3 voice. Must be lowercase and valid for the model. |
| `AGENT_SYSTEM_PROMPT` | No | `You are a helpful voice assistant. Keep responses short and conversational.` | System message seeded as the first entry of the conversation. |
| `HTTP_PORT` | No | `8000` | Port uvicorn binds on `0.0.0.0`. Also the port ngrok tunnels. |
| `PUBLIC_URL` | No | *(empty)* | Public HTTPS origin, trailing slash stripped. When set, no ngrok tunnel is started. When empty, `setup_ngrok()` runs. |
| `NGROK_AUTH_TOKEN` | No | *(empty)* | ngrok authtoken, applied only when `PUBLIC_URL` is empty. |
| `AGENT_WS_PORT` | No | `8001` | Port for the standalone WebSocket server in `agent.py`. Only used when `agent.py` is run directly; not present in `.env.example`. |

The following are compiled into `agent.py` rather than read from the environment.
Edit them in place to retune.

| Constant | Default | Effect |
|----------|---------|--------|
| `SILENCE_THRESHOLD` | 200 | RMS below this = silence |
| `SILENCE_FRAMES` | 40 | 40 × 20ms = 800ms silence → trigger STT |
| `MIN_SPEECH_FRAMES` | 8 | Ignore clips shorter than 160ms |

The LLM model (`gpt-4o-mini`), `max_tokens` (150), and `temperature` (0.7) are
also literals in `_llm()`.

## Running it

Start the server:

```bash
python server.py
```

You should see the banner with the resolved base URL:

```
============================================================
  Sarvam AI Agent
  Answer URL : https://<your-host>/answer
  Hangup URL : https://<your-host>/hangup
============================================================
```

Confirm the process is healthy and knows its own URL:

```bash
curl -s http://localhost:8000/health
# {"status":"ok","base_url":"https://<your-host>"}
```

Check the XML Vobiz will receive:

```bash
curl -s -X POST http://localhost:8000/answer
```

Now ring the number. The log should show, in order:

```
Call connected to AI agent
Stream started — id=<streamId>, call=<callUUID>
STT: <what the caller said>
LLM: <the model's reply>
Playback complete
```

The caller hears `नमस्ते! मैं आपकी कैसे मदद कर सकता हूं?` as soon as the stream
starts, then a reply after each pause.

To host the session without FastAPI, run the standalone WebSocket server instead
and point a `<Stream>` at `ws://<host>:8001`:

```bash
python agent.py
```

## Vobiz Stream reference

### XML produced

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Stream bidirectional="true" keepCallAlive="true"
            contentType="audio/x-mulaw;rate=8000"
            statusCallbackUrl="https://your-server/stream-status"
            statusCallbackMethod="POST">
        wss://your-server/ws
    </Stream>
    <Hangup/>
</Response>
```

> **Note:** URL goes as text content inside `<Stream>`, not as a `url=` attribute.

`contentType` is set explicitly because Vobiz otherwise sends `audio/x-l16;rate=8000`;
this example expects mu-law throughout.

### HTTP endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/answer` | Returns bidirectional Stream XML |
| WS | `/ws` | Handles audio pipeline per call |
| POST | `/stream-status` | Stream lifecycle callbacks |
| POST | `/hangup` | Call ended callback |
| GET | `/health` | Health check |

### WebSocket events

| Direction | Event | Handled / sent where |
|-----------|-------|----------------------|
| Vobiz → agent | `start` | Captures `streamId` and the call id (`callId`, `start.callId`, or `start.callUUID`), then plays the greeting. |
| Vobiz → agent | `media` | Base64 mu-law payload fed to the VAD. Dropped while a turn is being processed. |
| Vobiz → agent | `playedStream` | Clears `is_playing`; only fires because the agent sends a `checkpoint`. |
| Vobiz → agent | `clearedAudio` | Clears `is_playing` after a barge-in flush. |
| Vobiz → agent | `stop` | Logged. |
| Agent → Vobiz | `playAudio` | One per 160-byte mu-law chunk, with `contentType: audio/x-mulaw` and `sampleRate: 8000`. |
| Agent → Vobiz | `checkpoint` | Sent after the last chunk, named `tts-<n>` where `n` is the conversation length. |
| Agent → Vobiz | `clearAudio` | Sent with `streamId` to flush queued playback on barge-in. |

### Audio format chain

| Stage | Format | Where |
|-------|--------|-------|
| Vobiz → agent | mu-law, 8 kHz, mono, base64 in `media.payload` | requested by `contentType="audio/x-mulaw;rate=8000"` |
| Decode | raw mu-law bytes | `base64.b64decode(payload)` |
| VAD measurement | signed 16-bit linear PCM, 8 kHz | `audioop.ulaw2lin(chunk, 2)` then `audioop.rms(pcm, 2)` |
| Buffer | mu-law bytes (the PCM is used only for the RMS) | `self._audio_buf` |
| STT upload | RIFF WAV — 1 channel, 2-byte samples, 8000 Hz | `_mulaw_to_wav()`, multipart field `file` as `audio.wav` |
| STT request | `model=saaras:v3`, `language_code=$AGENT_LANGUAGE`, `mode=transcribe` | `POST https://api.sarvam.ai/speech-to-text` |
| LLM | text only | `gpt-4o-mini`, full conversation history |
| TTS request | `model=bulbul:v3`, `speech_sample_rate=8000`, `enable_preprocessing=true` | `POST https://api.sarvam.ai/text-to-speech` |
| TTS response | base64 WAV in `audios[0]` | already 8 kHz because the sample rate is requested |
| Normalise | 8-bit widened to 16-bit if needed, resampled to 8 kHz if needed | `audioop.lin2lin(pcm, 1, 2)`, `audioop.ratecv(pcm, 2, 1, sr, 8000, None)` |
| Encode | mu-law, 8 kHz | `audioop.lin2ulaw(pcm, 2)` |
| Agent → Vobiz | 160-byte chunks = 20 ms each, base64 | `playAudio` events |

The `ratecv` call is the only resampling step in the whole chain, and it is a
no-op in the normal case: Bulbul is asked for 8 kHz directly, so the returned WAV
already matches the wire format. It exists so a different sample rate — Sarvam's
own default is 22050 Hz — still plays correctly if you change
`speech_sample_rate`. Nothing is ever upsampled; the pipeline is 8 kHz narrowband
end to end.

## Indic languages and voices

One variable, `AGENT_LANGUAGE`, drives both ends of the speech loop: it is passed
as `language_code` to Saaras v3 and as `target_language_code` to Bulbul v3. Change
it and the agent listens and speaks in the new language without any other code
change.

Saaras v3 accepts 23 language tags:

`hi-IN` Hindi · `bn-IN` Bengali · `kn-IN` Kannada · `ml-IN` Malayalam ·
`mr-IN` Marathi · `od-IN` Odia · `pa-IN` Punjabi · `ta-IN` Tamil ·
`te-IN` Telugu · `en-IN` English (India) · `gu-IN` Gujarati · `as-IN` Assamese ·
`ur-IN` Urdu · `ne-IN` Nepali · `kok-IN` Konkani · `ks-IN` Kashmiri ·
`sd-IN` Sindhi · `sa-IN` Sanskrit · `sat-IN` Santali · `mni-IN` Manipuri ·
`brx-IN` Bodo · `mai-IN` Maithili · `doi-IN` Dogri — plus `unknown` for
auto-detection.

`TTS_SPEAKER` selects the Bulbul v3 voice. The default here is `anand`; other
documented v3 voices include `simran`, `priya`, `ishita`, `kavya`, `aditya`, and
`rohan`, with `shubh` as the model default. Speaker names are case-sensitive and
must be lowercase. Check the
[Sarvam documentation](https://docs.sarvam.ai/) for the current catalogue and for
which voices are available in which language before you switch.

Code-mixing is handled by the models rather than by this code. Saaras v3 exposes a
`mode` parameter — this example sends `transcribe`; `translate`, `verbatim`,
`translit`, and `codemix` are the other values, and changing the literal in
`_stt()` changes what the transcript looks like. Bulbul v3 accepts code-mixed
input text, which is why a Hinglish reply from the LLM synthesises cleanly.

**Three things to change together when you switch language.** `AGENT_LANGUAGE`
alone is not enough:

1. `TTS_SPEAKER` — pick a voice that suits the target language.
2. The greeting on the `start` event, which is a hardcoded Hindi string
   (`नमस्ते! मैं आपकी कैसे मदद कर सकता हूं?`), and the LLM error fallback in
   `_llm()`, also hardcoded Hindi.
3. `AGENT_SYSTEM_PROMPT` — the default prompt is English and says nothing about
   output language, so the LLM will often reply in English even when the STT
   transcript is Hindi. Tell it explicitly, for example: *"You are a helpful
   voice assistant. Always reply in Hindi. Keep responses short and
   conversational."*

## Latency and barge-in

Time-to-first-audio after the caller stops speaking is the sum of four things:

| Stage | Cost | Notes |
|-------|------|-------|
| VAD hold | **800 ms, fixed** | `SILENCE_FRAMES` (40) × 20 ms. Nothing starts until the detector is convinced the turn has ended. This is the largest fixed component and the first knob to turn. |
| Saaras v3 STT | one round trip | The whole utterance is uploaded as a single WAV and awaited; `httpx` timeout is 15 s. Longer utterances mean larger uploads. |
| GPT-4o-mini | one round trip | Non-streaming — `_llm()` awaits the complete response before returning. Capped at `max_tokens=150`, which bounds the worst case. |
| Bulbul v3 TTS | one round trip | Also non-streaming. The complete WAV is synthesised and returned before a single byte can be played. |
| Playback | negligible to start | `_play_audio()` writes every 20 ms frame back to back as fast as the socket accepts; Vobiz paces delivery to the caller. |

Lowering `SILENCE_FRAMES` shortens the pause before the agent answers but makes it
more likely to cut in mid-sentence when the caller pauses to think. Raising
`SILENCE_THRESHOLD` helps on noisy lines at the cost of clipping quiet speech.

**Barge-in.** `_process()` checks `is_playing` before running the pipeline and, if
the agent is mid-reply, sends `clearAudio` with the `streamId` to flush what Vobiz
has queued. `is_playing` is set when playback starts and cleared by `playedStream`
or `clearedAudio` — which is why the agent sends a `checkpoint` after the last
chunk, since Vobiz only emits `playedStream` in response to one.

Two behaviours follow from where that check sits, and both are worth knowing
before you tune:

- The flush happens **after** the VAD has segmented a complete utterance — so 800 ms
  after the interrupting caller stops speaking, not the instant they start. The
  agent finishes rather more of its sentence than a caller might expect.
- While `_processing` is true, inbound `media` events are dropped entirely. Audio
  spoken during the STT → LLM → TTS turn is not captured, so a caller who talks
  over the thinking pause will not be heard.

There is also no echo gating: the VAD runs on all inbound audio, including
whatever of the agent's own voice returns on the line.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Sarvam returns 401/403, or keys look empty despite a populated `.env` | `SARVAM_API_KEY` or `OPENAI_API_KEY` is unset, still holds a placeholder, or the `.env` was created somewhere other than beside `server.py` | Confirm both keys are filled in the `.env` next to `server.py`, or export them into the shell before starting |
| `ModuleNotFoundError: No module named 'audioop'` | Running on Python 3.13+, where `audioop` was removed from the standard library | Use Python 3.9–3.12, or install an `audioop` backport |
| Caller hears silence; log shows `Sarvam TTS 4xx` | Invalid `TTS_SPEAKER` for `bulbul:v3`, or a speaker/language mismatch. `_tts()` returns empty bytes and `_speak()` then skips playback without raising | Use a documented lowercase v3 speaker; the warning line includes the status and the first 200 characters of Sarvam's response body |
| Agent understands Hindi but replies in English | `AGENT_LANGUAGE` steers only Sarvam STT and TTS. The default `AGENT_SYSTEM_PROMPT` is English and specifies no output language | Add an explicit instruction to `AGENT_SYSTEM_PROMPT`, e.g. "Always reply in Hindi" |
| Vobiz never opens the WebSocket | `PUBLIC_URL` empty and ngrok not authorised, or the Application's answer URL does not match the printed base URL | `curl /health` to see the resolved `base_url`, then confirm the Application answer URL is `<base_url>/answer` with method POST |
| Log reads `Stream ended — UUID=…, status=?` | The `/stream-status` handler reads a `StreamStatus` form field; the Vobiz callback posts the lifecycle name in `Event` (alongside `CallUUID`, `StreamID`, `From`, `To`) | Read `form.get("Event")` — and `StreamID` — in `stream_status()` |
| Agent responds to its own voice, or talks over itself | No echo gating: `_handle_audio()` runs the VAD on every inbound frame, including audio returning on the line during playback | Gate the VAD on `is_playing`, or raise `SILENCE_THRESHOLD` above the echo floor for your carrier |
| Long silences, then a burst of replies | The caller pauses shorter than 800 ms, so utterances merge; or `MIN_SPEECH_FRAMES` (8 frames = 160 ms) is discarding short answers like "हाँ" | Lower `SILENCE_FRAMES`, and lower `MIN_SPEECH_FRAMES` if single-word answers are being dropped |

## Security notes

- **Two provider keys with real cost attached.** `SARVAM_API_KEY` and
  `OPENAI_API_KEY` are read from the environment and used server-side only —
  neither ever reaches the caller or the browser. The Sarvam key travels in the
  `api-subscription-key` header on every STT and TTS request. Keep `.env` out of
  version control; the repository has no `.gitignore`, so add one before your
  first commit.
- **Call audio leaves your infrastructure.** Every segmented utterance is uploaded
  to `api.sarvam.ai`, and the resulting transcript — plus the entire running
  conversation for that call — is sent to OpenAI on each turn. If you handle
  regulated or personal data, review both providers' retention and processing
  terms, and check whether your jurisdiction requires disclosure or consent
  before recording or processing a caller's voice.
- **Transcripts are written to the logs.** `_process()` logs the caller's words at
  `INFO` (`STT: …`) and the model's reply (`LLM: …`). Anything a caller says —
  including an account number or an address — lands in whatever collects stdout.
  Redact these lines or drop the level before running against real traffic.
- **Conversation state is per-process memory.** `self.conversation` grows for the
  life of the call and is discarded when the WebSocket closes. Nothing is
  persisted, which limits exposure, but also means the history is readable in a
  process dump.
- **The endpoints are unauthenticated.** `/answer`, `/stream-status`, `/hangup`,
  and the `/ws` upgrade accept any caller that can reach them. Terminate TLS in
  front of the app, restrict inbound traffic to Vobiz, and validate the callback
  parameters before trusting them.
- **ngrok is for development only.** `setup_ngrok()` publishes your local port on a
  public URL with no access control. Set `PUBLIC_URL` to a controlled origin for
  anything beyond testing.

## Roadmap

> Planned improvements to this example. Ideas and pull requests are welcome —
> open an issue to discuss anything here.

- [ ] **Broaden Indic-language coverage.** Move the hardcoded Hindi greeting and
      the `_llm()` fallback string into a per-language message table, and select
      the language per call from the `start` event rather than once per process.
- [ ] **Stream the TTS response.** `_tts()` currently awaits the complete WAV
      before the first `playAudio` frame is sent; streaming synthesis would cut
      time-to-first-audio noticeably.
- [ ] **Add a test suite.** There are no tests today — the VAD state machine, the
      mu-law ⇄ WAV conversions, and the event router are all pure enough to cover
      with fixtures and no network.
- [ ] **Persist conversations.** `self.conversation` lives only in the
      `CallSession` and is lost at hangup; writing transcripts to a store would
      enable review, analytics, and resuming a call.
- [ ] **Tune interruption handling.** Detect barge-in from the first speech frame
      instead of waiting for full VAD segmentation, add echo gating, and keep
      capturing `media` events while a turn is being processed.
- [ ] **Add retries and provider fallback.** `_stt()` and `_tts()` return empty on
      any error and `_llm()` returns a fixed apology; a retry with backoff and a
      secondary provider would make a transient failure inaudible.
- [ ] **Ship packaging and observability.** A pinned `requirements.txt`, a
      container image, and per-stage latency metrics would make the example
      deployable as-is.

## Contributing

Issues and pull requests are welcome. If you are changing the audio path, please
say which Python version you tested on — `audioop` behaviour is the usual source
of surprises — and describe how you verified the change on a real call.

Before opening a pull request:

```bash
python -m compileall server.py agent.py   # syntax check
python server.py                          # boots and prints the banner
curl -s -X POST http://localhost:8000/answer   # returns well-formed XML
```

Then place a test call and confirm the `start → STT → LLM → Playback complete`
sequence appears in the log. There is no automated test suite yet; adding one is
on the roadmap and a good first contribution.