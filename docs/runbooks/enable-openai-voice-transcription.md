# Runbook: Enable OpenAI Voice Transcription — deploy steps retired

**Deploy mechanism retired 2026-08-30, not the feature itself.** This
runbook's own step 2 (`ansible-playbook ansible/playbooks/home-agent.yml`/
`open-webui.yml`) belonged to `infra/home-infra`'s Docker-based deployment
of both services — both playbooks are gone, along with the roles' own
container management, once `home_agent`/`open_webui` moved to k3s-native
Deployments (`infra/k3s-apps`' `modules/home_agent`/`modules/open_webui`).
The transcription relay itself (`home_agent`'s `/v1/audio/transcriptions`
endpoint) is vendored into that same k3s-apps module and deployed through
its normal Terraform apply — no dedicated runbook for that mechanism.

**Still genuinely useful, unchanged by the deployment migration** — the
feature and its Open WebUI-side configuration didn't move, only how the
underlying services get deployed:

## Flip the setting Open WebUI already persisted (not optional)

**This step is easy to miss and the feature will silently keep failing
without it.** Open WebUI stores most settings, including which
speech-to-text engine is active, in its own database — seeded from
environment variables only the *first* time a setting has no DB row yet.

**This is not the same page as the per-chat Settings gear icon** — that
one only offers "Default" and "Web API" by design. The real engine picker
is in the separate Admin Panel, which only an admin account can reach:

1. Click your profile/username in the bottom-left of the sidebar →
   **Admin Panel**.
2. Inside the Admin Panel: **Settings** → **Audio**.
3. Under **Speech-to-Text Engine**, select **OpenAI**.
4. Selecting "OpenAI" auto-fills the API Base URL with the real
   `https://api.openai.com/v1` — you must change it yourself to:

   ```text
   http://home-agent:8000/v1
   ```

   The API Key field is required by the form but never actually checked
   (`home-agent` ignores it and uses its own real key server-side) — type
   any non-empty placeholder. If the STT Model field also complains,
   enter `whisper-1` — also cosmetic only. Save.

5. **If you speak a non-English language**: set it explicitly rather than
   relying on auto-detection, which is unreliable on short clips. This is
   a *personal* setting, not an Admin one — open the per-chat Settings
   gear icon → **Audio** and set **Language** to your language code
   (e.g. `de`). The relay forwards whatever you set here unchanged.

## Verify live

1. In a chat, click the microphone/voice-recording button and say a short
   test phrase.
2. Confirm it transcribes correctly into text.
3. Confirm a normal (non-voice) chat message still works, and that
   `ai.jkandler.de` overall is unaffected.

See [`current-state.md`](../current-state.md)'s ADR 0012
writeup for whether final German-transcription confirmation has actually
landed yet -- as of that doc's last update it was still pending.
