# Operations

Everyday commands. Procedures for specific tasks are in
[`runbooks/`](runbooks/README.md); what is live is in
[`current-state.md`](current-state.md). Never print secret values,
unfiltered `docker inspect` output or Kubernetes Secrets.

## Quick public checks

Without credentials (expected results checked 2026-10-11):

```bash
check() { curl -s -o /dev/null -m 10 -w "%{http_code} %{redirect_url}\n" "$1"; }
check https://nextcloud.jkandler.de/status.php        # 200
check https://auth.jkandler.de/api/health             # 200
check https://docs.jkandler.de/api/documents/         # 401 without a token
check https://stalwart.jkandler.de/.well-known/jmap   # 307 to /jmap/session
check https://www.jkandler.de/                        # 200
```

Grafana (`grafana.jkandler.de`) shows the same picture from Prometheus,
including blackbox probes and backup age. Logs are in Grafana's Explore
with the `Loki` datasource: `{job="host-journal"}` for the homeserver's
journal (`boot_id` selects one boot, e.g. the one before a crash; the
values match `journalctl --list-boots`), `{job="k3s-pods", namespace="apps"}`
for Pod logs, including Pods that no longer exist. Retention 30 days.

## Homeserver

```bash
ssh julian@192.168.178.100
docker ps --format '{{.Names}} {{.Status}}'   # Prometheus, exporters, Loki
systemctl list-timers nextcloud-restic.timer
systemctl --failed
systemctl list-timers backup-freshness.timer
df -h / /mnt/black-hdd /mnt/red-hdd
```

Nextcloud runs in k3s since 2026-11-21 (ADR 0033); `occ` is
`kubectl -n apps exec deploy/nextcloud -c nextcloud -- php occ …`. AIO is
parked on the host (stopped, kept for a rollback).

For "the homeserver died" use the `investigate-homeserver-outage` skill
and [`runbooks/homeserver-silent-freeze.md`](runbooks/homeserver-silent-freeze.md).

## k3s cluster

Headlamp, the read-only cluster UI, is at `k8s.jkandler.de` (forward-auth). For
`kubectl`, SSH to `k3s-node-1` through the homeserver (the VM network is
not reachable from the LAN). Workloads change only through
[`k3s-apps`](https://github.com/KandlerLi/k3s-apps) or [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap), never `kubectl apply`.

## Applying changes

Every change goes through code. The `roll-out-infra-change` skill has the
branch, validate and PR steps and the doc update; the
[change rollout view](architecture/current/dynamic-ansible.md) shows the
flow.

- **Argo CD-synced** ([`k3s-apps`](https://github.com/KandlerLi/k3s-apps)): merge the PR; Argo CD syncs `main`
  within a few minutes, ESO delivers Secrets from Secrets Manager.
- **CI-applied repos** (`aws/*`, [`stalwart-config`](https://github.com/KandlerLi/stalwart-config)): merge the PR; CI plans
  on the PR and applies on merge.
- **Platform Argo CD** ([`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)'s `platform/`, ADR 0037): merge the PR,
  wait for the OutOfSync push, then sync by hand on
  `argo-platform.jkandler.de` (LAN): `platform-root` first, then each changed
  child Application on its own page. Nothing syncs or prunes by itself.
- **Local Terraform roots** ([`repo-infra`](https://github.com/KandlerLi/repo-infra),
  [`terraform-state`](https://github.com/KandlerLi/terraform-state), [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)): `git pull` on
  `main` first (an apply from a stale checkout silently applies the old
  config), then `./scripts/roll-out.sh plan` and `apply` in that repo. Any
  unexpected create, replace or destroy is a stop condition.
- **Ansible** ([`home-infra`](https://github.com/KandlerLi/home-infra)): `scripts/roll-out.sh` (homeserver,
  `site.yml`) and `scripts/roll-out-k3s.sh` (k3s VMs, `k3s.yml`) run the
  tests, a dry run and the real apply with `--ask-become-pass`.
- **After merging a PR** in those repos or [`fritzbox`](https://github.com/KandlerLi/fritzbox):
  `scripts/update.sh` switches to a fresh `main` and runs the roll-out script
  (`plan`/`apply`; in `home-infra` `dry-run`/`apply` plus the venv and
  collections, `apply k3s` for `k3s.yml`).

## DNS

Records are Terraform-owned: subdomains in [`dyndns`](https://github.com/KandlerLi/dyndns)
`deployment.auto.tfvars`, mail records in [`ses-relay`](https://github.com/KandlerLi/ses-relay), `www` in
[`website`](https://github.com/KandlerLi/website). Change them by PR, never in the Route 53 console; verify
with a public resolver afterwards.

## Secrets

Values live in AWS Secrets Manager, one secret per group holding several
keys as JSON. Change a value by merging into the existing JSON from a
silent prompt, never by overwriting the whole secret or passing the value
as a command-line argument. Commands and rotation steps:
[`runbooks/rotate-secrets.md`](runbooks/rotate-secrets.md) and
[`secrets-manager`](https://github.com/KandlerLi/secrets-manager)'s `rotation-schedule.md`.

## Plain reboot

`sudo reboot` on the homeserver takes the k3s VMs down with it. Expect
SSH back after several minutes (most of it is the VMs' shutdown), then
~15 minutes until the public services answer and ~13 until all CI
runners are online: both VMs cold-start from the black HDD (measured
2026-10-25). Nextcloud comes back with `k3s-node-1`. The "CI – GitHub runners"
Grafana dashboard shows the runners recovering.

## Planned shutdown (hardware maintenance)

Copy [`home-infra/scripts/shutdown-homeserver-for-maintenance.sh`](https://github.com/KandlerLi/home-infra/blob/main/scripts/shutdown-homeserver-for-maintenance.sh)
to the host, run it with `sudo bash … --check`, then without `--check`, and
type `POWER OFF` when asked. It shuts down both k3s VMs (Nextcloud with
them), checks the disks and failed units, syncs and powers off. It does not take a backup.
After boot the VMs start by themselves; then run the public checks above.
