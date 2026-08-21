# ADR 0012: Relay speech-to-text through home-agent instead of local Whisper

- Status: Accepted
- Date: 2026-08-21
- Deployment status: Implemented, not yet deployed

## Context

The user tried to use Open WebUI's voice-recording feature and hit:

```
Error transcribing chunk: Cannot find an appropriate cached snapshot folder
for the specified revision on the local disk and outgoing traffic has been
disabled. To enable repo look-ups and downloads online, set
'HF_HUB_OFFLINE=0' as environment variable.
```

Open WebUI's default speech-to-text engine is a bundled local Whisper model
(via `faster-whisper`), which downloads its weights from Hugging Face Hub
on first use. Two things about this deployment specifically block that,
both deliberate, both confirmed against the pinned `v0.11.0` source rather
than assumed:

- `open_webui`'s container runs on the `home-agent-frontend` Docker network
  with `com.docker.network.bridge.enable_ip_masquerade: "false"`
  (`ansible/roles/home_agent/tasks/main.yml`) — no internet egress at all,
  by design, so the container it's confined to Nextcloud/home-agent only.
- `OFFLINE_MODE: "True"` is already set in `open_webui`'s env
  (`ansible/roles/open_webui/tasks/main.yml`). Reading
  `backend/open_webui/env.py` directly: when `OFFLINE_MODE` is true, Open
  WebUI sets `os.environ['HF_HUB_OFFLINE'] = '1'` itself — this is exactly
  where the error's `HF_HUB_OFFLINE` message comes from, and it was already
  intentional hardening against exactly this kind of unplanned phone-home
  (it also disables the version-update check and RAG model auto-update).

So the failure is expected given the existing security posture, not a
misconfiguration — voice input was simply never wired up for it.

Three approaches were considered with the user: browser-native
transcription (Web Speech API, sends audio to the browser's own speech
provider — e.g. Google, from the user's browser, not this server), routing
through `home-agent` to a real STT API, or disabling voice input entirely.
The user asked how ChatGPT itself does it for comparison: ChatGPT's browser
only records audio; the actual transcription always happens server-side, on
OpenAI's own infrastructure. The user chose the equivalent design here.

## Decision

`home-agent` — the one component in this stack with real internet
egress — gets a new relay endpoint, `POST /v1/audio/transcriptions`
(`ansible/roles/home_agent/files/agent/home_agent/audio.py`, wired into
`api.py`). Open WebUI's built-in `openai`-compatible STT engine is pointed
at it via `AUDIO_STT_ENGINE=openai`, `AUDIO_STT_OPENAI_API_BASE_URL`, and
`AUDIO_STT_OPENAI_API_KEY` (the same provider base URL and placeholder key
already used for chat), which sidesteps `OFFLINE_MODE` entirely rather than
loosening it — the `openai` STT engine never touches Hugging Face Hub, it
only makes an HTTP request to whatever base URL is configured.

The exact multipart request shape Open WebUI sends
(`backend/open_webui/routers/audio.py`'s `_transcribe_openai` in the pinned
source) was read directly, not guessed: a `multipart/form-data` POST with a
`file` field (the recorded audio) and a `model` field, to
`{api_base_url}/audio/transcriptions`. `audio.py`'s `_extract_audio_part`
parses the incoming body just far enough to pull out the `file` part
(bytes, filename, content type) using the stdlib `email` module — no new
dependency — then `OpenAIAudioTranscriber` builds a *fresh* outgoing
multipart request to OpenAI's real endpoint via `httpx2` (already a direct
dependency here, used elsewhere for Unix-socket tool calls), with the model
always set from `HOME_AGENT_STT_MODEL` (env var, default `whisper-1`) —
**never** from whatever the incoming request's `model` field said. This
mirrors the existing rule in the chat path (`OpenAIResponsesProvider`
always uses its own configured `HOME_AGENT_MODEL`, never a caller-supplied
one) rather than introducing a new trust exception for audio.

The upload is capped at 25 MB — OpenAI's own limit for this endpoint, so
there's no reason to accept more — checked from the `Content-Length` header
before the body is ever read, so an oversized claimed length is rejected
without buffering anything.

**One operational catch, not a code issue**: Open WebUI persists most
settings — including `audio.stt.engine` — into its own database on first
boot, and its `Config.seed_defaults()` only inserts keys that don't already
exist; "existing DB values take precedence over defaults"
(`backend/open_webui/models/config.py`, confirmed by reading it directly).
Since this instance already booted at least once with STT unset, the new
env vars alone will **not** retroactively flip the already-persisted `''`
(local Whisper) value — a one-time manual change in Open WebUI's own Admin
Settings → Audio page is also needed after deploying. See
`docs/runbooks/enable-openai-voice-transcription.md`.

## Consequences

- Recorded audio now leaves the homeserver, going to OpenAI via the same
  trusted path (and the same API key) chat already uses — no new external
  destination is introduced, and no new credential is issued.
- `home-agent`'s role changes slightly: previously it only ever executed a
  small, fixed set of server-decided tool calls (never relaying arbitrary
  caller-supplied binary content anywhere) or forwarded chat text to
  OpenAI's chat API. This is the first capability where it relays
  caller-supplied binary content to an external API — bounded by the same
  size cap OpenAI itself enforces, and by the model being pinned
  server-side so the caller can't redirect spend/behavior by naming a
  different model.
- `OFFLINE_MODE` and the frontend network's no-egress design are untouched
  and remain the intended posture for everything else Open WebUI might try
  to reach out to (RAG embeddings, version checks, any other
  Hugging-Face-backed feature) — this fix is scoped to STT only.
- The Admin Settings toggle needed after deploy is a real, easy-to-miss gap
  between "code deployed" and "feature actually works" — flagged explicitly
  in the runbook so it isn't rediscovered live.
