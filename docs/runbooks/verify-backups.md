# Runbook: Verify and Restore Backups

Every backup lives on `/mnt/red-hdd` on the homeserver (`current-state.md`,
"Backups"). This runbook has two parts: the **drill**, which proves the
backups are restorable without touching production, and the **restore**
procedures per backup for when they are needed.

Status: the drill passed on 2026-10-26 for all six checks then; the
`stalwart` check was added on 2026-10-28. The restore
procedures below have **not** been exercised end to end; each one restores
into production, so read it fully before starting.

## Drill: prove the backups are restorable

Run from the laptop, in [`home-infra`](https://github.com/KandlerLi/home-infra) (prompts for the homeserver's sudo
password):

```bash
aws login    # optional: without it the Authelia key check is skipped
scripts/verify-backups.sh
```

It copies the newest backups into a root-only scratch directory under
`/mnt/red-hdd`, checks them, deletes the scratch directory, and prints one
PASS/FAIL line per check:

| Check | Proves |
| --- | --- |
| `deluge-config` | the newest tarball extracts |
| `open-webui` | the newest tarball extracts and `webui.db` passes `PRAGMA integrity_check` |
| `authelia` | the newest backup decompresses to a sound SQLite database |
| `authelia-key` | the **current** storage encryption key decrypts it |
| `paperless` | every exported original exists and matches the checksum in `manifest.json` |
| `stalwart` | the newest tarball extracts and holds RocksDB's `CURRENT` and the generated config |
| `borg` | the passphrase opens AIO's Borg repository and a file comes back intact from the newest archive |

Secrets (Borg passphrase, Authelia key, sudo password) go to the homeserver
on stdin only. Run the drill after any change to a backup job, after a key
rotation, and before risky hardware work (the SSD migration in the task
list, a disk swap).

**Where the keys are:**

- Borg (Nextcloud AIO): `pass` entry `nc/backup` (verified by the drill on
  2026-10-26). AIO shows the passphrase only once at setup; without it the
  repository is unrestorable.
- Authelia: `authelia_storage_encryption_key` in Secrets Manager
  (`home-infra/authelia`). A backup is encrypted with the key that was live
  when it was taken. Backups from before a rotation need the previous key,
  which Secrets Manager keeps only as `AWSPREVIOUS` until the next write to
  that secret. The last rotation was 2026-10-25 22:37; Authelia backups are
  kept 14 days.

## Restore: Nextcloud (AIO Borg)

Into a fresh AIO instance (e.g. after a rebuild where `/mnt/black-hdd` was
lost). AIO's own restore flow does the work:

1. Bring up the mastercontainer (`site.yml`), open
   `https://192.168.178.100:8080` and choose to **restore a former AIO
   instance from backup**.
2. Backup location: `/mnt/red-hdd` (AIO appends `borg`; the repository is
   `/mnt/red-hdd/borg`). Passphrase: `pass show nc/backup`.
3. Pick the archive (newest unless the newest is the broken state) and let
   AIO restore. It restores the data, database and AIO configuration.
4. Check: log in, open a representative file, check the AIO interface
   shows all containers healthy.

## Restore: Authelia (TOTP, WebAuthn, consent)

The database lives in the PVC `authelia-data` in namespace `auth`
(local-path on `k3s-node-1`); the Pod mounts it at `/data/db.sqlite3`.
Steps 2 to 4 ran for real on 2026-10-27, when Authelia moved from
`default` to `auth` (tracker #32), with a copy of the live file instead
of a backup: TOTP login worked afterwards.

1. Pick the backup in `/mnt/red-hdd/authelia-backup`
   (`authelia-db-<YYYYmmdd-HHMMSS>.sqlite3.gz`). If it predates the last
   storage key rotation, set the Secrets Manager value back to the key
   from that time first, or re-encrypt the restored file with `authelia
   storage encryption change-key` afterwards.
2. Stop Authelia: `kubectl scale -n auth deployment authelia --replicas=0`
   (Terraform sets it back to 1 on the next `k3s-apps` apply).
3. On `k3s-node-1`, `gunzip -c` the backup over
   `/var/lib/rancher/k3s/storage/<authelia-data volume>_auth_authelia-data/db.sqlite3`
   (volume name: `kubectl get pvc -n auth authelia-data`), keeping the
   file's owner and mode (`nobody:nogroup`, `0644`). Remove any
   `db.sqlite3-wal`/`-shm` next to it first.
4. Start Authelia again (`kubectl scale -n auth ... --replicas=1` or re-run the
   `k3s-apps` apply) and check `authelia storage encryption check
   --config /config/configuration.yml` in the Pod reports `SUCCESS`, then
   log in with TOTP.

## Restore: Stalwart (mail)

`stalwart-backup` (`k3s-apps`, `modules/stalwart/backup.tf`) runs at
03:15: it scales Stalwart to 0, tars its whole volume (`etc/` with the
generated config, `data/` with the RocksDB store) to
`/mnt/red-hdd/stalwart-backup/stalwart-<YYYYmmdd-HHMMSS>.tar.gz` and
scales it back to 1. 14 days are kept. If it ever fails before scaling
up, `StalwartDown` fires after 15 minutes:
`kubectl -n apps scale deploy/stalwart --replicas=1`.

Restore into the live volume (`/var/lib/k3s-apps-data/stalwart` on
`k3s-node-1`, a static PV from `k3s-bootstrap`):

1. `kubectl -n apps scale deploy/stalwart --replicas=0` and wait until
   its Pod is gone.
2. Copy the tarball to `k3s-node-1` (from the homeserver: `sudo cat`
   it into `ssh ... 'sudo tee /root/stalwart.tar.gz'`).
3. On `k3s-node-1`: move the current `etc` and `data` aside, then
   `tar -xzf /root/stalwart.tar.gz -C /tmp/s` and copy
   `/tmp/s/stalwart/.` into `/var/lib/k3s-apps-data/stalwart/` with
   `cp -a`. The files belong to uid/gid 2000.
4. `kubectl -n apps scale deploy/stalwart --replicas=1`, then log in to
   Bulwark and check the newest mail is there.

## Restore: Paperless

`document_exporter` mirrors everything (documents, metadata, users,
settings) into `/mnt/red-hdd/paperless-export` nightly. Restoring means
importing that into an **empty** Paperless instance (fresh database and
media), per Paperless-ngx's `document_importer` documentation:

1. Bring up Paperless from [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) with empty `paperless-postgres` and
   `paperless-data` volumes and an empty `/mnt/black-hdd/paperless-media`.
2. Run `python3 manage.py document_importer <export dir>` in the Paperless
   container with the export mounted (the `paperless-export` PVC).
3. Check: document count matches the drill's count, open a few documents,
   search works.

## Restore: Deluge config and Open WebUI data

Dated tarballs in `/mnt/red-hdd/aux-backups`
(`deluge-config-<date>.tar.gz`, `open-webui-<date>.tar.gz`), each holding
the directory itself. Both directories are NFS-exported to the k3s Pods.

1. Scale the Deployment to 0 (`deluge` or `open-webui`).
2. As root on the homeserver, move the current directory aside and extract
   the tarball into its parent: `/mnt/black-hdd` for `deluge-config`
   (owner `deluge:deluge`), `/var/lib` for `open-webui` (owner
   `open-webui-container:root`). `tar` as root keeps owners and modes.
3. Scale back to 1 and check the UI (Deluge torrents, Open WebUI chats).
