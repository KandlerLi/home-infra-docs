# Runbook: Deploy and Publish Deluge

Status: **Deployed privately and verified; publication is the only step
left.** Three deploy attempts each surfaced a real, distinct bug (see step
2's history below — Deluge has no "disable login" mode, a config-schema
version mismatch, and finally a stop/rewrite race with the outgoing
container), each found only by independently verifying live state with
`docker exec`/`docker logs` rather than trusting a clean Ansible run. The
third fix is confirmed live: the default password is rejected by a direct
JSON-API check. Nextcloud wiring (step 3) is also done and verified — the
`occ files_external` JSON shape assumption held. Every remaining
`ansible-playbook` run below needs a human-entered become password.

This adds a BitTorrent client with a Traefik-fronted, Basic-Auth-gated web UI
at `torrent.jkandler.de`, with completed downloads visible from Nextcloud.
Production deployment and public publication are separate, deliberately
guarded, human-confirmed actions — not a single step.

## 1. Preflight

1. ✅ **Done.** Add `"torrent"` to `dyndns`'s `subdomains` set — merged in
   [dyndns#12](https://github.com/KandlerLi/dyndns/pull/12),
   `torrent.jkandler.de` resolves.
2. ✅ **Done.** Two independent credentials, both generated and written via
   `sops set` — no plaintext value was written to this repository or ever
   printed to a terminal/chat, and each was shown to the user once, out of
   band, to save in their password manager:
   - Traefik Basic Auth (the primary public gate): a random 32-byte
     password and its bcrypt hash (`$2b$`, 12 rounds) as
     `shared_ingress_deluge_auth_password`/`_password_hash`.
   - Deluge's own web UI login: **this cannot be disabled** — confirmed by
     reading `deluge/ui/web/auth.py` directly, a missing `pwd_sha1` makes
     `check_password()` return `False` for every password (permanently
     locking out login, not bypassing it), so Deluge has no "no auth" mode
     at all. It gets its own independent random password,
     `deluge_web_password`, pre-seeded into `web.conf` before the
     container's first start (Deluge doesn't write a default config to
     disk on its own). You will see **two separate login prompts** —
     Traefik's, then Deluge's — not one.
3. Run unit tests and syntax checks:

   ```bash
   cd /home/julian/projects/infra/home-infra
   .venv/bin/python -m unittest discover -s tests -v
   .venv/bin/ansible-playbook ansible/playbooks/site.yml --syntax-check
   ```

## 2. Deploy Deluge privately

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/deluge.yml --ask-become-pass
```

This creates the dedicated `deluge` service account, `/mnt/black-hdd/downloads`
and `/mnt/black-hdd/deluge-config`, pulls the pinned image, and starts the
container with the web UI bound to `127.0.0.1:8112` only.

Before starting the container, the role pre-seeds `/config/web.conf`
(`{{ deluge_config_dir }}/web.conf`) with `deluge_web_password`'s hash,
computed with Deluge's own algorithm (`sha1(salt + password)`, verified
against `auth.py`'s `_change_password()` and cross-checked locally against
Python's `hashlib` before use), written at schema `"file": 2` with every
key from Deluge's actual `CONFIG_DEFAULTS` (matching `DelugeWeb`'s
`ConfigManager(file_version=2)` exactly, so no version-migration path runs
— an earlier attempt at `"file": 1` triggered one, though that migration
function turned out only to touch the `language` key, not the password).
The check for whether this still needs to happen is content-based: it
looks for the known default `pwd_sha1` value and re-seeds if found, so it
self-repairs rather than treating "a file is present" as already correct.

**The real root cause of the password reverting to Deluge's default,
found after the schema fix above still didn't hold**: a still-running
`deluge-web` process holds its own in-memory copy of `web.conf` and
periodically autosaves it. If the outgoing container's process is still
alive (e.g. mid-way through Docker's stop grace period) when the template
task writes a freshly seeded `web.conf`, that process's next autosave
silently overwrites the fix with its own stale, default-holding in-memory
state — before the recreated container ever starts. This was confirmed
live by comparing `web.conf.bak` (which Deluge's own `Config.save()`
creates before every resave) against `web.conf`: the backup held the
correct, freshly generated password, timestamped seconds before the
default-holding file that replaced it, and *both* writes predated the
recreated container's own start time — proving the previous container's
process, not the new one, clobbered the fix. The role now stops the old
container first, before writing `web.conf`, closing that race.

Verify locally before continuing:

```bash
ssh -L 8112:127.0.0.1:8112 julian@192.168.178.100
```

Open `http://127.0.0.1:8112`, log in with the `deluge_web_password` value
(check your password manager), and confirm the UI loads. Add a test torrent
(e.g. a Linux ISO) and confirm it downloads into `/mnt/black-hdd/downloads`.

## 3. Wire downloads into Nextcloud

Only after step 2 is confirmed working:

1. Decide which Nextcloud account (your own personal login, not a service
   account) should see the downloads folder.
2. Add to `ansible/inventory/group_vars/all/main.yml`:

   ```yaml
   nextcloud_aio_mount_dir: /mnt/black-hdd/downloads
   nextcloud_aio_mount_applicable_user: "<your-nextcloud-username>"
   ```

3. Apply:

   ```bash
   .venv/bin/ansible-playbook ansible/playbooks/site.yml --ask-become-pass
   ```

   This sets `NEXTCLOUD_MOUNT` on the AIO mastercontainer (which **recreates
   it** — expect brief AIO unavailability) and registers the folder as
   Nextcloud "Local" external storage via `occ files_external:create`.

**Verified live**: `occ files_external:list --output=json`'s ID field is
indeed named `mount_id` — the assumption the "Record the downloads mount
id" task was built on held. Confirmed against the deployed state: mount id
`2`, mount point `/Downloads`, `datadir` pointed at
`/mnt/black-hdd/downloads`, `applicable_users: ["admin"]`.

4. Confirm the "Downloads" folder appears in Nextcloud's file list for the
   configured account, and that the test torrent from step 2 is visible and
   readable. (Backend registration is verified via `occ`; the actual
   Nextcloud web UI file list still needs a human to look at it.)

**This mount is not covered by AIO's built-in backup** — see
`docs/storage.md`.

## 4. Publish the web UI

Only after steps 2 and 3 are both independently confirmed:

```bash
.venv/bin/ansible-playbook ansible/playbooks/publish-deluge.yml \
  --ask-become-pass \
  --extra-vars deluge_publish_confirmation=PUBLISH_DELUGE
```

Verify:

1. `https://torrent.jkandler.de/` prompts for Basic Auth before showing
   anything.
2. The correct credential (from step 1) grants access to the Deluge UI.
3. An incorrect or missing credential is rejected.
4. Existing routes (`ai.jkandler.de`, `nextcloud.jkandler.de`) are unaffected.

Once verified, persist the deployed state:

```yaml
# ansible/inventory/group_vars/all/main.yml
deluge_enabled: true
shared_ingress_deluge_enabled: true
```

## 5. Inbound peer connectivity (manual, out-of-band)

BitTorrent peer connections can't be proxied through Traefik — they're a raw
TCP/UDP protocol, not HTTP. `deluge_peer_port` (default `6881`, both TCP and
UDP) is published on the Docker host's real network interface, not loopback,
so it can be reached — but reaching it from the internet also needs a port
forward on the FRITZ!Box router to the homeserver's LAN IP. This is not
Ansible-managed, same as the existing DynDNS FRITZ!Box client configuration.
Torrenting still works without this (outbound connections and any peer able
to reach you), just with reduced connectability.

## 6. Roll back the route

```bash
.venv/bin/ansible-playbook ansible/playbooks/rollback-deluge.yml \
  --ask-become-pass \
  --extra-vars deluge_rollback_confirmation=ROLL_BACK_DELUGE
```

This removes the public route only; the Deluge container, its downloads, and
its config are left intact.
