# Runbook: Deploy and Publish Deluge

Status: **Implemented (`deluge` role, shared_ingress route, nextcloud_aio
integration), not yet deployed.** Every `ansible-playbook` run below needs a
human-entered become password and has not been run against the homeserver.
Several implementation details (marked below) were not verified against a
live container and may need a follow-up fix on the first real run.

This adds a BitTorrent client with a Traefik-fronted, Basic-Auth-gated web UI
at `torrent.jkandler.de`, with completed downloads visible from Nextcloud.
Production deployment and public publication are separate, deliberately
guarded, human-confirmed actions — not a single step.

## 1. Preflight

1. ✅ **Done.** Add `"torrent"` to `dyndns`'s `subdomains` set — merged in
   [dyndns#12](https://github.com/KandlerLi/dyndns/pull/12),
   `torrent.jkandler.de` resolves.
2. ✅ **Done.** Basic Auth credential for Deluge, independent of the legacy
   API's credential: a random 32-byte password and its bcrypt hash
   (`$2b$`, 12 rounds) were generated and added via `sops set` as
   `shared_ingress_deluge_auth_password`/`_password_hash` in
   `ansible/inventory/group_vars/all/secrets.sops.yml` — no plaintext value
   was written to this repository or ever printed to a terminal/chat.
   **The password itself was shown to the user once, out of band, to save
   in their password manager; it is not repeated here or anywhere else.**
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

**Known unverified step**: the role waits for Deluge to generate its default
`web.conf` on first boot, then strips the `pwd_sha1` line to disable Deluge's
own login prompt (Traefik Basic Auth is the sole intended gate). The exact
config path
(`{{ deluge_config_dir }}/.config/deluge/web.conf`) was not confirmed against
a live container. **If the "Wait for Deluge's web.conf" task fails**, find
the real path with `docker exec deluge find /config -name web.conf`, fix the
path in `ansible/roles/deluge/tasks/main.yml`, and re-run. Until this step
succeeds, Deluge's own login is still active with its **default weak
`admin`/`deluge` credential** — low risk while the web UI is loopback-only,
but fix it before publishing publicly.

Verify locally before continuing:

```bash
ssh -L 8112:127.0.0.1:8112 julian@192.168.178.100
```

Open `http://127.0.0.1:8112` and confirm the web UI loads without a login
prompt (or, if the pwd_sha1 step didn't run yet, log in once with the
default credential and change it from Preferences → Interface → Password as
a stopgap). Add a test torrent (e.g. a Linux ISO) and confirm it downloads
into `/mnt/black-hdd/downloads`.

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

**Known unverified step**: the exact JSON shape of
`occ files_external:list --output=json` (specifically whether each entry's
ID field is named `mount_id`) was not confirmed against a live Nextcloud
version. **If "Record the downloads mount id" fails**, run
`docker exec nextcloud-aio-nextcloud php occ files_external:list --output=json`
by hand, find the real field name, and fix
`ansible/roles/nextcloud_aio/tasks/main.yml`.

4. Confirm the "Downloads" folder appears in Nextcloud's file list for the
   configured account, and that the test torrent from step 2 is visible and
   readable.

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
