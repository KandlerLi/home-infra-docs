# Runbook: Open WebUI voice transcription

Speech-to-text goes from Open WebUI to `home_agent`'s relay (ADR 0012),
which forwards it to OpenAI's Whisper with its own key. Open WebUI's side
is set entirely by env vars in
[`k3s-apps/argocd/open-webui/deployment.yaml`](https://github.com/KandlerLi/k3s-apps/blob/main/argocd/open-webui/deployment.yaml)
(`AUDIO_STT_ENGINE=openai`, `AUDIO_STT_OPENAI_API_BASE_URL=http://home-agent-svc/v1`).
The API key there is a placeholder that `home_agent` ignores, and so is
the model, which `home_agent` pins server-side.

There is no Admin Panel step. Since 2026-11-15 Open WebUI ignores settings
stored in its database (`ENABLE_PERSISTENT_CONFIG=False`, see
[decisions](../decisions.md#open-webui-takes-its-settings-from-env-only)).
Changes made under Admin Panel → Settings → Audio last only until the next
Pod restart.

## Language

Auto-detection is unreliable on short clips. Set the language per user:
the Settings gear icon → **Audio** → **Language** (for example `de`). This
is a user setting, stored per account and unaffected by the above. The
relay forwards it unchanged.

## Verify live

1. In a chat, record a short phrase with the microphone button.
2. Check that it transcribes correctly.
3. Check that a normal text message still works.
