# Runbook: Rebuild the Homeserver

Status: **outline, never exercised end to end.** The order below follows
the current setup (2026-10-11). Steps marked *open* have no written
procedure yet (tracked in the private task list); do not improvise them under
pressure without reading the tool's own restore documentation first.

## Preconditions

- Console access to the machine and Debian 13 installation media.
- GitHub access to the repos (all code is there; nothing on the server is
  the only copy of any code).
- AWS access as the IAM user `julian` (MFA): Ansible and Terraform read
  every secret from Secrets Manager, so this replaces any local key.
- The disks: `black-hdd` (live data) and `red-hdd` (backups). If
  `black-hdd` is lost, the data comes back from `red-hdd`; if both are
  lost, there is no further copy (no off-site backup, see the private task list).

## Safety constraints

- Install the OS on the system SSD only. Confirm each disk's identity and
  filesystem UUID before mounting; never format either HDD as part of a
  rebuild.
- Never recursively change ownership below `/mnt/black-hdd` or
  `/mnt/red-hdd`.
- Make sure the HDD mount is in place before Docker or libvirt start, so an
  empty mount point cannot hide a failed mount.

## Order

1. Install Debian 13 on the SSD, create user `julian`, enable SSH.
2. On the laptop: clone [`home-infra`](https://github.com/KandlerLi/home-infra), set up its venv as in its
   README ("First apply"), log in to AWS, check `ansible home_servers -m
   ansible.builtin.ping`.
3. Apply `site.yml` (`scripts/roll-out.sh`). The `storage` role mounts both
   disks by UUID; verify with `findmnt /mnt/black-hdd /mnt/red-hdd` and a
   look at the expected top-level directories (`docs/storage.md`).
4. Nextcloud: if `black-hdd` survived, AIO picks up its data directory and
   volumes; otherwise restore from the Borg repository on
   `/mnt/red-hdd/borg` through the AIO interface. *Open:* written restore
   steps, and where the Borg repository's passphrase is kept.
5. Apply `k3s.yml` (`scripts/roll-out-k3s.sh`): both k3s VMs, k3s itself.
   The VM disks live on `black-hdd`; if they survived, the cluster comes
   back with its local-path volumes.
6. Apply [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap) locally: PersistentVolumes, RBAC and the
   CI runners. CI cannot run before this step.
7. Re-run the `Apply` workflow of [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) on `main` (workflow
   dispatch) to recreate every workload.
8. Restore k3s data that lived only in the VMs, if the VM disks were lost
   (*open*: written steps for each):
   - Authelia (TOTP/WebAuthn registrations): `/mnt/red-hdd/authelia-backup`
   - Paperless: `document_exporter` output in `/mnt/red-hdd/paperless-export`
     (Paperless media itself is on `black-hdd`)
   - Open WebUI data and Deluge config live on the host and are in
     `/mnt/red-hdd/aux-backups`
   - Stalwart mail has no backup outside the VM.
9. Check DNS (the FRITZ!Box keeps updating the apex record), certificates,
   `systemctl --failed`, the Prometheus targets and backup-age alerts in
   Grafana, and the public checks in `docs/operations.md`.

## Done when

A representative Nextcloud file opens, every public check in
`docs/operations.md` gives its expected result, Grafana shows all targets
up, and the next nightly backups have run.
