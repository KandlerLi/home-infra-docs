# Runbook: Close Open WebUI's native-login MFA-bypass gap

Status: **Fixed and confirmed live on 2026-09-11.** This is a data-level fix,
not a Terraform change — see "Why this isn't in Terraform" below for why, and
re-read this runbook if it ever needs to be redone.

Applies to the current k3s deployment of Open WebUI
([`k3s-apps/modules/open_webui`](https://github.com/KandlerLi/k3s-apps/blob/main/modules/open_webui), `kubernetes_deployment_v1.open_webui`,
pod label `app=open-webui`) — not the older homeserver-Docker deployment
described in `publish-open-webui.md`, which this cluster deployment
superseded.

## Background

Authelia's ingress-level gate only proves you reached a valid Authelia
session (MFA required to get one). Once past it, Open WebUI's own native
email/password login was a second, completely independent credential path
that skipped MFA entirely — a real bypass, not a theoretical one.

`modules/open_webui/main.tf` sets three env vars meant to close this:
`ENABLE_LOGIN_FORM=False` (hide the UI), `ENABLE_PASSWORD_AUTH=False` (the
actual protocol-level switch — blocks password auth at the backend
regardless of UI), `ENABLE_SIGNUP=False` (blocks new native accounts). These
were applied and the Deployment rolled out cleanly, but the login form kept
rendering at `https://ai.jkandler.de/auth`.

## Root cause

Confirmed live by reading Open WebUI's own SQLite database
(`/app/backend/data/webui.db`, a `config` table of flat `key`/`value`/
`updated_at` rows — Open WebUI's "PersistentConfig" mechanism). These
settings are read from environment variables **only on a container's
first-ever launch**, then written to this table and used from there on every
subsequent start — a later change to the env var alone does nothing once a
row already exists.

This deployment's database predates the `ENABLE_LOGIN_FORM`/
`ENABLE_PASSWORD_AUTH`/`ENABLE_SIGNUP` env vars (it was carried over from the
original homeserver-Docker deployment, not created fresh under this
Terraform module). So on 2026-09-11 the persisted rows were:

- `ui.enable_login_form` = `true` — stale, never updated. **This is what
  actually caused the form to keep rendering.**
- `ui.enable_signup` = `false` — already correct; matched the env var by
  coincidence of prior configuration, nothing to fix.
- Password auth was independently confirmed live to be genuinely blocked at
  the backend (a real login attempt returned "restricted as a security
  measure", not a UI-only hide) — `ENABLE_PASSWORD_AUTH` was working
  correctly despite the same PersistentConfig mechanism, so **the actual
  security property already held**; the login form itself was a cosmetic
  leftover, not a live bypass, by the time this was found and fixed.

Also confirmed: `ENABLE_OAUTH_SIGNUP=True` (still on) is unrelated and
correctly left on — it only allows a *new* Open WebUI account to be
provisioned after a successful Authelia (MFA-gated) login for a user Open
WebUI hasn't seen before. It is not a native-login path. `ENABLE_SIGNUP`
(native self-registration) is the one that matters here, and was already
off.

There is no Admin Panel toggle for `ui.enable_login_form` in this Open WebUI
version (checked live, both Admin → Settings → Authentication and →
General) — it is genuinely only reachable by editing the database directly
or by a from-scratch first boot.

## The fix (performed 2026-09-11)

Read-only diagnosis, then a single scoped `UPDATE`, run from an interactive
SSH session on the k3s node itself — not nested inside a single remote
command string, and not through a local `kubectl`.

Local `kubectl` binaries (laptop or homeserver) fail against this cluster's
k3s v1.36 with `couldn't get current server API group list ... BadRequest` —
a client/server version-skew issue, not a permissions one. The reliable
workaround is to SSH to the node and use its own bundled kubectl, which
always matches the server version:

```bash
ssh -J julian@192.168.178.100 ansible@192.168.101.10
```

Then, on the node:

```bash
# 1. confirm the actual persisted value first -- don't guess the key name
sudo k3s kubectl exec deploy/open-webui -- python3 -c '
import sqlite3
con = sqlite3.connect("/app/backend/data/webui.db")
for row in con.execute("SELECT * FROM config WHERE key LIKE \"ui.enable%\""):
    print(row)
'
# (Open WebUI's image has no sqlite3 CLI binary -- use its own bundled
# Python instead, via python3 -c.)

# 2. back up the live db before writing anything
sudo k3s kubectl exec deploy/open-webui -- cp \
  /app/backend/data/webui.db /app/backend/data/webui.db.bak-<date>

# 3. the actual write -- touches only this one row
sudo k3s kubectl exec deploy/open-webui -- python3 -c '
import sqlite3, time
con = sqlite3.connect("/app/backend/data/webui.db")
con.execute("UPDATE config SET value = ?, updated_at = ? WHERE key = ?",
            ("false", int(time.time()), "ui.enable_login_form"))
con.commit()
print("rows changed:", con.total_changes)
'
# expect: rows changed: 1

# 4. restart so the running process re-reads config on boot
#    (PersistentConfig values are cached in memory -- a raw SQL edit is
#    invisible to the already-running process until it restarts)
sudo k3s kubectl rollout restart deployment/open-webui
sudo k3s kubectl rollout status deployment/open-webui
```

Verify: load `https://ai.jkandler.de/auth` in a fresh/private tab. Only
"Sign in with Authelia" should be visible — no email/password fields.
Confirmed live 2026-09-11.

The `webui.db.bak-<date>` backup is left inside the pod's data volume
(`/app/backend/data`, the NFS-backed mount from `storage.tf`) — safe to
leave for now, worth deleting once the fix is confirmed stable over time
(nothing sweeps it automatically).

## Why this isn't in Terraform

There is no Terraform (or Admin API / Admin UI) path to a Open WebUI
PersistentConfig row — the only interfaces are a first-ever container boot
(env vars) or the Admin Panel (and this particular setting has no Admin
Panel control in this version). This is exactly the same category as
Nextcloud's OIDC client-secret update in `rotate-secrets.md` — a deliberate,
documented, human-run one-off against live application state, not
infrastructure, so it sits outside the "changes go through Terraform/
Ansible" rule for this workspace. Re-run the steps above by hand if it ever
needs to be redone; don't wrap it in a script.

## When this could regress

- **A genuinely fresh deployment** (new PVC, no carried-over `webui.db`)
  would pick up `ENABLE_LOGIN_FORM=False` correctly on first boot and would
  never need this fix — the gap only exists because this instance's database
  predates that env var.
- **This same database surviving into a future rebuild** (PVC preserved,
  container recreated) keeps the fixed row — no regression expected from a
  routine redeploy or image upgrade, since `terraform apply` only changes
  the Deployment spec, not the mounted data volume.
- If the login form ever reappears at `/auth`, re-run step 1 above to check
  `ui.enable_login_form` before assuming anything else — it's the first
  thing to suspect, and confirmed cheap to check.
