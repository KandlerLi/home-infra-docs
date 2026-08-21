# Runbook: Enable OpenAI Voice Transcription

Status: **Implemented, not yet deployed.** See ADR 0012 for the design and
why local Whisper can't work in this deployment.

Fixes: `Error transcribing chunk: Cannot find an appropriate cached
snapshot folder for the specified revision on the local disk and outgoing
traffic has been disabled...` when recording voice in Open WebUI.

## 1. Run tests and syntax checks

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/ansible-playbook ansible/playbooks/site.yml --syntax-check
```

## 2. Deploy

Both `home-agent` (the new `/v1/audio/transcriptions` relay) and
`open-webui` (the new `AUDIO_STT_*` env vars) need to be redeployed:

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/home-agent.yml --ask-become-pass
.venv/bin/ansible-playbook ansible/playbooks/open-webui.yml --ask-become-pass
```

## 3. Flip the setting Open WebUI already persisted (not optional)

**This step is easy to miss and the feature will silently keep failing
without it.** Open WebUI stores most settings, including which
speech-to-text engine is active, in its own database — seeded from
environment variables only the *first* time a setting has no DB row yet.
Since this instance already booted with STT unset (that's the whole reason
this bug exists), the new `AUDIO_STT_ENGINE=openai` env var will **not**
retroactively override the already-persisted local-Whisper setting on its
own.

**This is not the same page as the per-chat Settings gear icon** — that
one only offers "Default" and "Web API" by design (it lets a regular user
defer to the server config or force their own browser's engine, nothing
more; "Default" there is already correct and needs no change). The real
engine picker is in the separate Admin Panel, which only an admin account
can reach:

1. Click your profile/username in the bottom-left of the sidebar →
   **Admin Panel**.
2. Inside the Admin Panel: **Settings** → **Audio**.
3. Under **Speech-to-Text Engine**, you should see five real options —
   OpenAI, Web API, Deepgram, Azure AI Speech, MistralAI. Select
   **OpenAI**.
4. **Found live: these are not pre-filled the way this runbook originally
   assumed.** Selecting "OpenAI" auto-fills the API Base URL with the real
   `https://api.openai.com/v1` — you must change it yourself to:

   ```text
   http://home-agent:8000/v1
   ```

   The API Key field is required by the form but never actually checked
   (`home-agent` ignores it and uses its own real key server-side) — type
   any non-empty placeholder, e.g. `home-agent-internal`. If the STT Model
   field also complains, enter `whisper-1` — also cosmetic only. Save.

5. **If you speak a non-English language**: set it explicitly rather than
   relying on auto-detection, which is unreliable on short clips (found
   live: German speech sometimes came back transcribed as English). This
   is a *personal* setting, not an Admin one — open the per-chat Settings
   gear icon → **Audio** (the same page that only showed Default/Web API
   earlier) and set **Language** to your language code (e.g. `de`). The
   relay forwards whatever you set here unchanged.

## 4. Verify live

1. In a chat, click the microphone/voice-recording button and say a short
   test phrase.
2. Confirm it transcribes correctly into text — no
   `HF_HUB_OFFLINE`/"outgoing traffic has been disabled" error, and no
   `413, message='Content Too Large'` error (that was a second bug, found
   live and fixed: Open WebUI streams the recording as a
   `Transfer-Encoding: chunked` body with no `Content-Length` at all, which
   an earlier version of the relay treated as zero-length and rejected —
   make sure you're on a revision at or after that fix).
3. Confirm a normal (non-voice) chat message still works, and that
   `ai.jkandler.de` overall is unaffected.
4. Optional, read-only: `docker logs home-agent` should show no errors for
   the `/v1/audio/transcriptions` request; it never logs audio content or
   the transcribed text, only the exception type on failure (matching this
   repo's existing privacy-safe logging convention).

## 5. Roll back

Set `AUDIO_STT_ENGINE` back to unset (or explicitly `""`) in
`ansible/roles/open_webui/tasks/main.yml`'s env block and redeploy — but
because of the same DB-persistence behavior as step 3, you'll also need to
flip the Admin Settings dropdown back manually (to local Whisper, which
still won't work here, or another configured engine) rather than relying
on the env var alone. To fully remove the relay endpoint itself, revert
`home_agent/audio.py` and its wiring in `api.py`.
