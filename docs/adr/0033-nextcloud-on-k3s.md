# ADR 0033: Move Nextcloud from AIO on the host to k3s

- Status: Accepted (2026-11-21)
- Date: 2026-11-18
- Deployment status: in progress; test instance (migration step 2) in review
- Supersedes: [ADR 0003](0003-nextcloud-aio.md)

## Context

Nextcloud is the last large production service running directly on the
homeserver instead of in k3s. It runs as Nextcloud All-in-One (ADR 0003):
an Ansible-managed mastercontainer that starts and updates its sibling
containers (Apache, Nextcloud, PostgreSQL, Redis, Imaginary, Collabora,
Talk, Notify Push, Whiteboard) through the Docker socket. The data
directory is `/mnt/black-hdd/nextcloud`, and AIO's Borg backs it up
nightly together with the database into `/mnt/red-hdd/borg`.

Most of what sits around it already lives in k3s:

- The public route `nextcloud.jkandler.de` goes through k3s Traefik
  (a fixed route to `192.168.101.1:11000`). If `k3s-node-1` is down,
  Nextcloud is already unreachable from outside.
- SSO goes through Authelia in k3s (the `nextcloud` OIDC client).
- `sankey-export` writes into Nextcloud from a k3s CronJob.

What AIO costs today:

- It is the only image pinned to `:latest`. Its sub-containers update
  themselves outside Git and outside the Trivy image scan.
- It needs a workaround: `nextcloud-aio-apache-network-fix.service`
  reattaches Apache to its Docker network after reboots (cause unknown,
  inside AIO; see `decisions.md`).
- It cannot run in Kubernetes. The mastercontainer needs the Docker
  socket.
- Config hides in AIO's Docker volumes and in the mastercontainer UI
  (`:8080`) rather than in Git. AIO also remembers state such as
  `NEXTCLOUD_MOUNT` after the variable is removed.

Moving to k3s does not remove Docker from the host: the monitoring
stack (Prometheus, node_exporter, cAdvisor, blackbox_exporter) still
runs there.

Options considered:

- **Keep AIO (status quo).** Works, needs least effort, and updates
  itself. Keeps the costs above.
- **Official Nextcloud Helm chart** (`nextcloud/helm`). Covers app,
  cron and an optional database. But nothing in `k3s-apps` uses Helm
  today. It would bring Argo CD Helm rendering and a large values
  surface for a single-user instance.
- **Plain manifests in `k3s-apps`**, like Paperless and Vaultwarden:
  the official `nextcloud` image (Apache variant), PostgreSQL, Redis and
  a cron CronJob. Matches every other workload.

## Decision

Run Nextcloud in k3s as plain manifests under
`k3s-apps/argocd/nextcloud`, digest-pinned and deployed by Argo CD.
Retire AIO once the cluster instance has taken over.

**Components.**

- `nextcloud` (official image, Apache variant), one replica, reachable
  through the existing Traefik route (the fixed route becomes a normal
  Service-backed IngressRoute).
- PostgreSQL on a node-local static PV under `/var/lib/k3s-apps-data`,
  same pattern as Paperless Postgres, not on NFS.
- Redis for caching and transactional file locking. Required, since the
  data directory sits on NFS.
- Background jobs as a CronJob running `cron.php` every 5 minutes.
- **Euro-Office** document server: the only AIO extra in use (enabled
  in the AIO UI, not in Ansible). It moves along as its own Deployment
  with a JWT secret from Secrets Manager, reached by Nextcloud's
  connector app through Traefik. The exact image and connector app are
  confirmed in migration step 1.
- Notify Push and Imaginary: optional, added only if needed after the
  move.
- Collabora, Talk and Whiteboard are not used and are not moved.

**Storage.** The data directory stays where it is,
`/mnt/black-hdd/nextcloud`, NFS-exported to `k3s-node-1` only, with
`root_squash` (`nfs_server` role). The pod runs as the uid/gid that owns
the directory on the host (`www-data`, 33), so no ownership change is
needed. `config/` and `custom_apps/` live on a node-local PV. Nothing
under `/mnt/black-hdd` is reorganised.

**Backup.** AIO's Borg goes away with AIO. In its place:

- A nightly CronJob writes `pg_dump` of the database and a copy of
  `config.php` to `/mnt/red-hdd/nextcloud-backup` over NFS.
- The files themselves are backed up on the host by **restic**, run by
  a systemd timer from `home-infra`, into a repository under
  `/mnt/red-hdd/nextcloud-restic`. Running it on the host avoids reading
  the whole data directory through NFS. The timer runs after the
  database dump, so both belong to the same night. Restic was picked
  over Borg because it can later write the same repository format
  straight to S3. That gives a direct path for the open off-site item,
  without needing a Borg server.
- The existing `/mnt/red-hdd/borg` repository stays readable until the
  new backup has been restored once successfully.

**Upgrades.** The image digest is bumped through Renovate PRs, like
every other image. Nextcloud upgrades must not skip a major version.
Major bumps are therefore done one at a time and by hand, after a fresh
database backup. Minor and patch bumps can merge normally.

**Auth.** Unchanged. The same Authelia `nextcloud` OIDC client, set up
through `user_oidc` with the same forced-SSO setting. App passwords and
WebDAV (sync clients, `sankey-export`) keep working without Authelia.

## Migration

1. Record the exact Nextcloud version AIO runs (the cluster instance
   must start on it), the Euro-Office image and connector app, and the
   installed Nextcloud apps.
2. Build the cluster instance in parallel, at a test hostname, on an
   empty data directory. Check SSO login, WebDAV, file locking under
   NFS, that cron runs, and that a document opens in Euro-Office.
   Measure the instance's memory on `k3s-node-1`. Give the VM more
   memory before cutover only if that measurement shows the need.
3. Cutover in a maintenance window:
   1. Put AIO into maintenance mode and take a final AIO Borg backup.
   2. Dump AIO's PostgreSQL and restore it into the cluster Postgres.
   3. Carry over `config.php` and fix the database host, Redis and
      trusted proxies.
   4. Point the cluster pod at the existing data directory.
   5. Switch the Traefik route and run `occ maintenance:repair` and
      `occ files:scan --all`.
4. Confirm live: sync clients reconnect without a fresh setup;
   `sankey-export` succeeds; the nightly backups land; one test restore
   of the new backup works.
5. Remove AIO with the `nextcloud_aio` role (mastercontainer, network
   fix service, ports). Its Docker volumes and `/mnt/red-hdd/borg` are
   deleted only after a separate, explicit decision.

Rollback until step 5: stop the cluster pod, switch the route back,
take AIO out of maintenance mode. AIO's database is untouched by the
migration. Files written to the shared data directory during the
cluster phase are picked up again with `occ files:scan`.

## Consequences

- Nextcloud is managed the same way as every other app: Git, Argo CD,
  digest pins, Trivy, Renovate. The `:latest` exception, the Apache
  network workaround and the `192.168.101.1:11000` binding go away.
- Updates stop being automatic. Major upgrades become a manual,
  recurring task.
- File I/O goes over NFS. With many small files this is slower than a
  local bind mount, and the Redis file locking is required.
- Nextcloud and Euro-Office memory moves into `k3s-node-1` (3 vCPU,
  10 GiB), which currently has headroom. If it gets tight, the VM gets
  more memory. The host has room for that once AIO's containers are
  gone.
- Backup becomes two pieces (database CronJob plus host-side restic)
  instead of AIO's single Borg job. Both need monitoring in Grafana and
  Alertmanager.
- Short downtime at cutover.
- Off-site backup stays a separate open item for everything on
  `red-hdd`. Restic makes it a later change of repository target, not a
  tool change.

## Implementation notes

Decided while building (2026-11-21):

- **Background jobs run in the Pod**, as a second container calling
  `cron.php` every 5 minutes, not as a CronJob. PostgreSQL and Redis stay
  on localhost, and no second Pod mounts the node-local code volume.
- **Rootless as `www-data` (33).** Apache binds port 80 through the
  namespaced sysctl `net.ipv4.ip_unprivileged_port_start`, which
  PodSecurity `restricted` allows. Redis is configured through `occ` in a
  start hook, because the image's `REDIS_HOST` handling writes into a
  root-owned PHP directory.
- **The data directory is mounted at `/mnt/ncdata`**, the path AIO uses.
  Nextcloud's root storage id contains that path.
- **Euro-Office is deferred.** AIO's image runs `supervisord` and
  `nginx` as root with its own PostgreSQL, RabbitMQ and Redis, which
  `apps` (`restricted`) does not admit. Nextcloud moves without it; a
  rootless image or an alternative is a separate decision. Until then
  there is no document editing after the cutover.
- **notify_push comes along from the start**; Imaginary only if
  previews (such as HEIC) are missing.
- Before the migration, about 240,000 `filecache` rows of two unmounted
  storages left over from earlier mounts were removed (after a database
  dump), so the dump that moves is smaller.
