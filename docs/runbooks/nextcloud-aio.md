# Runbook: Operate Nextcloud AIO

## Enable the Borg backup

`/mnt/red-hdd` (`storage` role, `red_hdd_*` vars) is a dedicated second
HDD for exactly this. AIO's backup path is configured entirely through its
own web interface -- there is no `NEXTCLOUD_DATADIR`-style environment
variable for it, so this step can't be automated in Ansible; confirmed
against AIO's own upstream docs before writing this rather than guessed.

1. Confirm the mount first: `ssh julian@192.168.178.100 'findmnt /mnt/red-hdd'`.
2. Open the AIO interface: `https://192.168.178.100:8080`.
3. Under the **Backup** section, set the backup directory to `/mnt/red-hdd`.
4. Click **Create Backup**.
5. **Save the shown encryption password somewhere durable and separate from
   this server** (e.g. `pass`) -- without it, the backup is unrestorable.
   AIO only shows it once.
6. If the first backup fails with a permission error, `/mnt/red-hdd` is
   `root:root`/`0755` by default (`storage` role) -- adjust
   `red_hdd_mode`/`red_hdd_owner` and re-apply rather than
   `chmod`-ing the live host directly.
7. Once the first backup succeeds, enable **Daily backups** from the same
   section. Default retention is AIO's own
   `--keep-within=7d --keep-weekly=4 --keep-monthly=6` -- fine as a
   starting point, adjustable later via `BORG_RETENTION_POLICY` on the
   mastercontainer if needed.

This covers Nextcloud user data and AIO's own configuration/database state.
What it does **not** cover (Deluge downloads, k3s data) and how those are
backed up instead: the Backups table in `docs/current-state.md`.

## Current model

Ansible manages the AIO master container. The master container manages the
Nextcloud, Apache, PostgreSQL, Redis, Collabora, Talk, Imaginary, Notify Push,
and Whiteboard containers. Traefik in k3s terminates TLS for
`nextcloud.jkandler.de` and routes it to AIO Apache on `192.168.101.1:11000`
(the homeserver's address on the k3s network; Apache also listens on
`127.0.0.1:11000`).

## Read-only health check

```bash
ssh julian@192.168.178.100
docker ps
df -hT / /mnt/black-hdd
systemctl --failed
```

Expected baseline:

- all ten AIO containers are running
- every observed AIO health status is healthy
- `/mnt/black-hdd` is mounted
- Traefik owns TCP ports 80/443 and is healthy
- AIO Apache publishes only `127.0.0.1:11000`
- AIO management ports 8080/8443 and Talk port 3478 match the architecture

## Before an Ansible change

1. Verify the HDD mount.
2. Review changes to image, ports, volumes, and `NEXTCLOUD_DATADIR`.
3. Confirm the AIO configuration volume name is unchanged.
4. Take or verify an appropriate backup.
5. Ensure no unrelated task changes ownership below `/mnt/black-hdd`.

## Never perform as routine cleanup

- delete `nextcloud_aio_*` volumes
- recreate the master configuration volume
- move the data directory
- expose the Docker socket to another application
- add an AAAA record based only on the presence of a host IPv6 address
- move ports 80/443 away from the `k3s_ingress_forward` DNAT to k3s Traefik

## Known issues

- **Apache loses its Docker network connection after some reboots.**
  Confirmed on two independent reboots: `nextcloud-aio-apache` came
  back up disconnected from the `nextcloud-aio` Docker network, unable
  to reach the other AIO containers, needing a manual `docker network
  connect nextcloud-aio nextcloud-aio-apache` to recover. The
  underlying startup race is unknown and owned by AIO's own
  mastercontainer, not something this repo can fix directly.
  `nextcloud-aio-apache-network-fix.service` (`ansible/roles/
  nextcloud_aio`) automates the recovery step instead: it runs on
  every boot and on every Ansible apply (`state: restarted`, not
  `started`, so a oneshot already `active (exited)` from a prior run
  still re-executes and self-heals a currently-dropped connection),
  and exits immediately as a no-op if Apache is already connected.
- **The mounted downloads external-storage mount's permission cache
  goes stale.** Nextcloud caches per-file permissions (including
  whether the delete button shows up at all) from the last scan of a
  mount -- it does not re-check the real filesystem live on every page
  load. Files under the downloads mount are written and deleted by
  Deluge entirely outside Nextcloud's awareness, and host-side
  permission changes to that directory don't trigger a rescan on their
  own either. Confirmed live: after fixing the host ACL so `www-data`
  could actually delete, the UI still showed no delete option -- even
  from a fresh, uncached browser profile -- until the mount was
  explicitly rescanned (`occ files:scan`). `ansible/roles/
  nextcloud_aio`'s own task runs this on every apply, not just once --
  it's cheap, and it's the only thing keeping the mount's cache from
  drifting from what Deluge does to the files underneath it.

## Reverse-proxy operation

AIO runs in its supported reverse-proxy mode
(`nextcloud_aio_reverse_proxy_enabled: true`); Apache is reached only by
Traefik in k3s over `192.168.101.1:11000`. Turning reverse-proxy mode off
would make AIO try to own ports 80/443 itself; do not change it during an
ordinary apply.

## Shut down for physical hardware maintenance

Use the tracked helper instead of cutting power or manually stopping individual
containers. The helper checks that `/mnt/black-hdd` is mounted, refuses to
continue while a libvirt VM is running (so the k3s VMs have to be shut down
first, see below), asks AIO to stop its application
containers, verifies the stop, flushes filesystem buffers, and requests an OS
poweroff. It does not create a backup.

First shut down the k3s VMs; this takes every k3s service (all public
hostnames except Nextcloud, LAN DNS, mail) offline until the host is back.
They start again by themselves at boot (`autostart`).

```bash
ssh -t julian@192.168.178.100 \
  'sudo virsh shutdown k3s-node-2 && sudo virsh shutdown k3s-node-1'
ssh -t julian@192.168.178.100 'sudo virsh list'   # repeat until empty
```

Then, from the controller:

```bash
cd home-infra

scp scripts/shutdown-homeserver-for-maintenance.sh \
  julian@192.168.178.100:/tmp/

ssh -t julian@192.168.178.100 \
  'sudo bash /tmp/shutdown-homeserver-for-maintenance.sh --check'
```

Verify that users have finished active uploads and that an appropriate recent
backup exists. Then start the guarded shutdown and type the exact confirmation
`POWER OFF`:

```bash
ssh -t julian@192.168.178.100 \
  'sudo bash /tmp/shutdown-homeserver-for-maintenance.sh'
```

If the script reports an error after beginning the AIO stop, do not cut power.
Follow its warning, inspect the remaining containers, and either rerun the
script after resolving the problem or start the application containers again
through the AIO interface.

After the chassis is fully off, disconnect mains power and discharge standby
power before opening it. Following the hardware work, boot the host and verify:

```bash
free -h
systemctl --failed
docker ps
curl --fail-with-body https://nextcloud.jkandler.de/status.php
```

The 2026-08-19 RAM maintenance confirmed that the deliberately stopped AIO
application containers remained stopped after the host boot. Start them through
the AIO interface at `https://192.168.178.100:8080`, then repeat the checks.
