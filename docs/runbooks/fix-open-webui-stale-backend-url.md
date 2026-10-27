# Runbook: Fix Open WebUI's stale home-agent backend URL

Status: **Found and fixed live 2026-09-12.** Same underlying mechanism as
`harden-open-webui-login.md` — read that runbook first for the full
explanation of *why* this class of bug exists (Open WebUI's PersistentConfig
settings seed from environment variables only on a container's truly
first-ever boot, then live entirely in its own SQLite database from then
on, permanently ignoring any later change to the env var). This runbook
covers a second, independent incident of the same mechanism, not the same
bug.

## Symptom

Open WebUI (`ai.jkandler.de`) loads, but no model can be selected — the
model dropdown is empty or errors.

## Root cause

Confirmed live by reading Open WebUI's own `config` table (same
`/app/backend/data/webui.db`, same `python3 -c 'import sqlite3; ...'`
inspection pattern as the login-form fix): two `PersistentConfig` rows
still held the *original Docker-era* backend URL —

```text
openai.api_base_urls          = ["http://home-agent:8000/v1"]
audio.stt.openai.api_base_url = "http://home-agent:8000/v1"
```

`http://home-agent:8000` was a Docker network alias, valid only for the
original homeserver-Docker deployment (see `modules/open_webui/main.tf`'s
own header comment: "Points at home_agent's own in-cluster Service ...
instead of the Docker \"home-agent\" network alias"). Since the k3s
migration (2026-08-30), [`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' own Terraform has correctly set
both of these via `OPENAI_API_BASE_URL`/`AUDIO_STT_OPENAI_API_BASE_URL` to
the real in-cluster Service address, `http://home-agent-svc/v1` — but
because this deployment's database was carried over from the Docker
deployment rather than started fresh, PersistentConfig never picked up
either corrected value. `home-agent-svc` (confirmed via `kubectl get svc`)
doesn't resolve to anything under the bare name `home-agent` at all — a
real DNS failure, not just a wrong port — so every model-list request from
Open WebUI's own pod failed outright, regardless of how long this had
actually been broken versus simply unnoticed.

Confirmed directly against `home-agent`'s own `/v1/models` endpoint (both
from inside its own Pod via `localhost:8000`, and from Open WebUI's Pod via
the *correct* `home-agent-svc` address) that the backend itself was healthy
the entire time — this was purely a stale client-side config value, not an
outage on `home-agent`'s side.

## The fix (performed 2026-09-12)

Same shape as the login-form fix: a real backup, a scoped two-row `UPDATE`,
a restart, then verification via a real HTTP request — not a script, a
deliberate one-time action from an interactive SSH session on the k3s node
(`ssh -J julian@192.168.178.100 ansible@192.168.101.10`, `sudo k3s kubectl`
for the same client/server version-skew reason documented in the
login-form runbook).

```bash
# 1. back up first
sudo k3s kubectl -n apps exec deploy/open-webui -- cp \
  /app/backend/data/webui.db /app/backend/data/webui.db.bak-<date>

# 2. the actual write -- touches only these two rows
sudo k3s kubectl -n apps exec deploy/open-webui -- python3 -c '
import sqlite3, time
con = sqlite3.connect("/app/backend/data/webui.db")
now = int(time.time())
con.execute("UPDATE config SET value = ?, updated_at = ? WHERE key = ?",
            ("[\"http://home-agent-svc/v1\"]", now, "openai.api_base_urls"))
con.execute("UPDATE config SET value = ?, updated_at = ? WHERE key = ?",
            ("\"http://home-agent-svc/v1\"", now, "audio.stt.openai.api_base_url"))
con.commit()
print("rows changed:", con.total_changes)
'
# expect: rows changed: 2

# 3. restart so the running process re-reads config on boot
sudo k3s kubectl -n apps rollout restart deployment/open-webui
sudo k3s kubectl -n apps rollout status deployment/open-webui
```

Verified for real, not just a clean apply: from inside Open WebUI's own
Pod, a request to `http://home-agent-svc/v1/models` with the same
`Authorization: Bearer home-agent-internal` header the app itself uses
returned `HTTP 200` with the real model list (`home-agent`), and the
`config` table's own rows now read `http://home-agent-svc/v1`.

## Why this isn't in Terraform

Identical reasoning to the login-form runbook: there is no Terraform,
Admin API, or Admin UI path to a PersistentConfig row once it's been set —
only a genuinely fresh first boot, or editing the database directly. A
brand-new deployment (fresh PVC, no carried-over `webui.db`) would never
hit this, since it would correctly seed from the env vars Terraform already
sets right.

## If this pattern recurs

Any Open WebUI setting Terraform manages via an env var is a candidate for
this same class of bug on this deployment specifically, since its database
predates several of them. If a similar symptom shows up again (a Terraform
value that's demonstrably correct in `main.tf`, but the running app
behaves as if it isn't), check the `config` table for that setting's own
key before assuming anything else — `SELECT key, value FROM config WHERE
key LIKE '%<hint>%' OR value LIKE '%<hint>%'` is the fast way to find it,
same pattern used both times this bug has actually been found.
