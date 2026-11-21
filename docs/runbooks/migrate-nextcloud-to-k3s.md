# Runbook: Move Nextcloud from AIO to k3s

The cutover of [ADR 0033](../adr/0033-nextcloud-on-k3s.md) (migration step 3):
AIO's database and `config.php` move into the k3s instance, which then
serves `nextcloud.jkandler.de` on the same data directory. Nothing is
copied on disk: `/mnt/black-hdd/nextcloud` stays where it is. Downtime
about 1–2 hours, mostly `files:scan`.

Machines: 💻 laptop, 🖥️ homeserver. Every `kubectl` runs on 💻.

## Before the window

All of these are done and checked a day ahead:

1. Merged and rolled out:
   - `home-infra` "export the live data directory, restic backs it up"
     (`scripts/update.sh apply`),
   - `k3s-bootstrap` "PV for the live data directory"
     (`scripts/roll-out.sh apply`),
   - `k3s-apps` "cutover prep" (hooks pause in maintenance mode, restore
     Job).
2. restic's first full run of the live directory succeeded: no
   `BackupStale`/`BackupMissing` for `nextcloud-restic`, and
   `restic snapshots` shows `/mnt/black-hdd/nextcloud`.
3. AIO's nightly Borg backup of the same night succeeded.
4. **Versions match.** AIO updates itself; the restore Job refuses to run
   when they differ:

   💻
   ```bash
   ssh julian@192.168.178.100 docker exec -u www-data nextcloud-aio-nextcloud php occ status | grep versionstring
   grep -o 'nextcloud:[0-9.]*-apache' /home/julian/projects/infra/k3s-apps/argocd/nextcloud/deployment.yaml
   ```

   If AIO is ahead, bump the image in `k3s-apps` first (same patch
   version, never across a major).
5. The two window PRs are open as drafts: `k3s-apps` "Nextcloud cutover"
   and `home-infra` "Park Nextcloud AIO".

## In the window

### 1. Silence the expected alerts

ContainerDown (AIO's containers), ServiceUnreachable (the blackbox probe
of `nextcloud.jkandler.de`) and sankey-export's failing runs fire while
Nextcloud is down. Silence them for 3 hours in Alertmanager:

💻
```bash
kubectl -n monitoring port-forward svc/alertmanager-svc 9093:9093
```

then <http://localhost:9093>, "New Silence".

### 2. Both instances into maintenance mode

💻
```bash
kubectl -n apps exec deploy/nextcloud -c nextcloud -- php occ maintenance:mode --on
ssh julian@192.168.178.100 docker exec -u www-data nextcloud-aio-nextcloud php occ maintenance:mode --on
```

From here on cron in both does nothing and clients get 503.

### 3. AIO's final dump and `config.php` into the backup directory

The restore Job reads both from `/mnt/red-hdd/nextcloud-backup` (`www-data`,
`0700`). They go through a private temp file in julian's home, because
the target is only writable as `www-data`:

💻
```bash
pass show hosts/homeserver/sudo | head -1 | \
  ssh julian@192.168.178.100 'set -e; read -r pw; s(){ printf "%s\n" "$pw" | sudo -S -p "" "$@"; }
    umask 077; T=$(mktemp -d)
    docker exec nextcloud-aio-database sh -c "pg_dump -U nextcloud -d nextcloud_database -Fc" > "$T/aio-final.dump"
    docker exec nextcloud-aio-nextcloud cat /var/www/html/config/config.php > "$T/aio-config.php"
    docker exec -i nextcloud-aio-database pg_restore -l < "$T/aio-final.dump" > /dev/null && echo "dump readable"
    s install -o www-data -g www-data -m 0600 "$T/aio-final.dump" "$T/aio-config.php" /mnt/red-hdd/nextcloud-backup/
    rm -rf "$T"; echo "copied"'
```

AIO's database itself is not touched: it is the rollback.

### 4. Park AIO

Mark the `home-infra` "Park Nextcloud AIO" PR ready, merge it, then:

💻
```bash
cd /home/julian/projects/infra/home-infra
git checkout main && git pull --ff-only
scripts/update.sh apply
```

The master and every AIO application container are stopped and stay
stopped across reboots.

🖥️ `docker ps --filter name=nextcloud-aio` lists nothing.

### 5. Switch the k3s instance to the live data

Mark the `k3s-apps` "Nextcloud cutover" PR ready and merge it. Argo CD
syncs within 3 minutes. The Pod restarts in maintenance mode (the hooks
skip themselves), now on `nextcloud-files`:

💻
```bash
kubectl -n apps rollout status deploy/nextcloud
kubectl -n apps exec deploy/nextcloud -c nextcloud -- ls /mnt/ncdata
```

`appdata_oc05tti5k1jb` must be in the listing (AIO's instance).

### 6. Restore AIO's database and `config.php`

💻
```bash
cd /home/julian/projects/infra/k3s-apps
git checkout main && git pull --ff-only
# A finished Job of the same name would make apply a no-op ("unchanged"): delete it first.
kubectl -n apps delete job nextcloud-restore-aio --ignore-not-found
kubectl apply -f argocd/nextcloud/cutover/restore-aio.yaml
kubectl -n apps wait --for=condition=complete --timeout=30m job/nextcloud-restore-aio
kubectl -n apps logs job/nextcloud-restore-aio -c restore-db
kubectl -n apps logs job/nextcloud-restore-aio -c config
```

`restore-db` replaces the k3s database with AIO's dump; `config` writes
AIO's `config.php` with the Pod's database and Redis settings, AIO-only
keys removed and maintenance mode on, and saves the test instance's as
`config.php.test-instance`. It stops if AIO's version differs from the
image's.

### 7. Bring Nextcloud up

💻
```bash
X(){ kubectl -n apps exec deploy/nextcloud -c nextcloud -- "$@"; }
X php occ maintenance:mode --off
# AIO's apps without code here (Talk, Whiteboard, Euro-Office, AIO's own).
X php occ app:disable spreed whiteboard eurooffice office nextcloud-aio
X php occ maintenance:repair --include-expensive
X php occ db:add-missing-indices
X /docker-entrypoint-hooks.d/before-starting/10-settings.sh
X /docker-entrypoint-hooks.d/before-starting/20-apps.sh
X php occ files:scan --all
X php occ notify_push:setup https://nextcloud.jkandler.de/push
X php occ status
```

`files:scan` walks the 109,000 files over NFS and takes the longest.

Rehearsed on 2026-11-21 against the test instance with a real AIO dump:
the restore Job took 22 s (225 tables), every enabled app's version in
AIO's database matched the code in the Pod except the four this step
disables, and the commands above (without `files:scan`) took 35 s with
no errors. The rehearsal ran on the empty test directory, so it couldn't
time `files:scan`.

### 8. Check

- Browser: `https://nextcloud.jkandler.de` → Authelia → own files, the
  existing account (same uid as in AIO).
- Desktop and phone client: sync resumes without a new setup.
- `notify_push:setup` above passed all checks.
- sankey-export: the next run (every minute) succeeds,
  `kubectl -n apps get jobs -l app=sankey-export`.
- `X php occ setupchecks`: nothing but known items.

Then remove the silence.

### 9. The next morning

- `nextcloud-backup` (03:30 UTC) and `nextcloud-restic` (03:45 UTC)
  succeeded: Grafana's Backups dashboard, no `BackupStale`.
- Delete the two cutover files, which hold AIO's salts:

  💻
  ```bash
  pass show hosts/homeserver/sudo | head -1 | \
    ssh julian@192.168.178.100 'sudo -S -p "" rm /mnt/red-hdd/nextcloud-backup/aio-final.dump /mnt/red-hdd/nextcloud-backup/aio-config.php'
  ```

## Rollback

Possible until AIO is removed (migration step 5). AIO's database was
never written to.

1. 💻 `kubectl -n apps exec deploy/nextcloud -c nextcloud -- php occ maintenance:mode --on`
2. Revert the `k3s-apps` cutover PR: the route goes back to AIO's
   Apache, the Pod back to the test hostname and directory.
3. Revert the `home-infra` park PR and roll it out: the master runs again.
4. In AIO's interface (port 8080) start the containers.
5. 💻 `ssh julian@192.168.178.100 docker exec -u www-data nextcloud-aio-nextcloud php occ maintenance:mode --off`
6. Files written while k3s served them are on disk, but AIO's database
   doesn't know them: `occ files:scan --all` in AIO.

## Afterwards (migration steps 4 and 5)

Separate PRs once the instance has run for a while:

- Remove the test instance's leftovers: PVC `nextcloud-data` and its PV,
  `/mnt/black-hdd/nextcloud-k3s-test` with its export, the
  `nextcloud-test` subdomain.
- Remove AIO (`nextcloud_aio` role, the AIO backup-window alert rule).
  Its Docker volumes and `/mnt/red-hdd/borg` only after a separate,
  explicit decision.
- Docs: `current-state.md`, `storage.md`, the AIO runbook to historical,
  the Nextcloud section of [Verify and restore backups](verify-backups.md),
  secrets-manager's `home-infra/nextcloud` runbook (no Ansible consumer
  any more), the `check-security-invariants` Nextcloud rules (integrations
  reach `nextcloud-svc`, not AIO's endpoint).
- Delete the pre-cleanup database dump in julian's `~/backups` on the
  homeserver (2026-11-21).
