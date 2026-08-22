# ADR 0015: Bring the Finanzfluss Sankey exporter into Ansible

- Status: Accepted
- Date: 2026-08-22
- Deployment status: Implemented, not yet deployed

## Context

`/home/julian/sankey` on the homeserver was a working, hand-maintained
script (`finanzfluss_export.py`, its own local-only git repo, a systemd
oneshot unit) that downloads a household budget spreadsheet from
Nextcloud, builds a `finanzfluss.de` Sankey-diagram URL from the parsed
income/expenses, drives headless Chromium via Playwright to intercept
that page's own "Download PNG" Highcharts export, and uploads the
resulting PNG back to Nextcloud. It only ran when triggered by hand
(`systemctl start sankey-export.service`) and lived entirely outside this
repo — no code review, no tests, no recovery if the host were rebuilt.

The user asked for it to be brought under this repo's normal
Ansible/git/runbook flow, cleaned up, kept WebDAV-only for Nextcloud
access, and automated so editing the spreadsheet produces a fresh diagram
without a manual trigger.

Live inspection of the script and its unit found three things worth
fixing along the way, all confirmed with the user via AskUserQuestion
before implementation:

1. It authenticated to Nextcloud as the **`admin`** account itself, over
   the **public** `https://jkandler.dedyn.io` URL, running as the
   `julian` Linux user.
2. Its secret lived in `systemd-creds`
   (`/etc/credstore.encrypted/sankey-nextcloud.cred`) — tied to that one
   machine, not recoverable from git.
3. No automation trigger existed.

## Decisions

**Keep the Playwright/Highcharts approach.** The user explicitly chose
this over switching to a native Python chart library: it produces the
exact finanzfluss.de visual style, and the interception logic
(`download_png_via_menu` in `finanzfluss_export.py`) is already hardened
against a real production failure (Highcharts' public export server
returning HTTP 429) — not something to discard for a rewrite.

**Dedicated Nextcloud account, loopback endpoint.** A new `sankey-export`
account (bootstrapped the same way as `nextcloud_tools`' own account —
see ADR 0008) is shared only `Documents/Finanzen`, with edit permission.
The script talks to `127.0.0.1:11000` with an explicit `Host:
nextcloud.jkandler.de` header, the same virtual-hosting requirement
`nextcloud_tools_service.py` already handles, instead of the public
dyndns URL.

**Generated app password, not SOPS.** The user's own instinct was to
align this with the rest of the project's secret handling, and picked
SOPS by analogy with `deluge_web_password`. But a Nextcloud app
password/token **cannot be pre-chosen** — `occ user:auth-tokens:add`
always generates it; there is no value to encrypt into SOPS ahead of
time the way there is for a login password a human picks. The actual
closest sibling in this repo is `nextcloud_tools`' own account:
generated once via `occ`, written to a host-only `0400` file
(`ansible/tasks/generate_and_validate_nextcloud_app_password.yml`,
consumed via `read_app_password()`), never in SOPS. That still delivers
what the user actually wanted — no more hand-managed, machine-only
secret — because Ansible owns the whole lifecycle end to end and can
regenerate it any time by rerunning `sankey-export.yml`. **Do not
"fix" this back toward SOPS** without re-reading this reasoning; there is
no app-password value a human could put in SOPS in the first place.

**Runs natively via systemd, not Docker.** This is the first
non-containerized service in this repo. Playwright's own Docker images
are large, the existing venv+systemd approach already worked, and there
is no real benefit to forcing it into a container just for consistency.
Hardening follows this repo's usual systemd-hardening posture
(`NoNewPrivileges`, `ProtectSystem=strict`, dedicated unprivileged
account, `CapabilityBoundingSet=` empty) with one deliberate exception:
Chromium launches with `--no-sandbox` unconditionally, not just as a
root fallback. This browser only ever visits one fixed, self-generated
finanzfluss.de URL — never arbitrary content — so the sandbox buys little
here, and skipping it avoids Chromium's own namespace/setuid sandbox
setup fighting with the unit's hardening (a source of exactly the kind
of unpredictable failure the user was worried about when asking for this
to stay simple).

**Polling, not push, at 1 minute — kept cheap on purpose.** Nextcloud has
no simple always-available webhook-on-file-change without meaningfully
more infrastructure (a new listener service, network reachability from
the AIO container network, signature verification), which didn't seem
justified for a personal automation. Instead, every run does a cheap
WebDAV `PROPFIND` for just the workbook's ETag first; the expensive
Playwright export only runs on the (rare) tick where the ETag actually
changed, cached in `/var/lib/sankey-export/last-etag`. This is what
keeps a 1-minute timer cheap rather than complex: 1,440 checks/day are
one small XML response each.

## Consequences

- `Documents/Finanzen` must be shared with the new `sankey-export`
  account manually, in the Nextcloud UI, before anything can run — same
  reasoning as every prior Nextcloud integration in this repo (ADR 0008):
  automating a share grant would need the folder owner's own credential.
- The blast radius of a leaked `sankey-export` app password is now one
  shared folder, not full Nextcloud admin.
- A failed Playwright export leaves the cached ETag stale on purpose, so
  the next tick retries automatically rather than needing a manual
  re-trigger.
