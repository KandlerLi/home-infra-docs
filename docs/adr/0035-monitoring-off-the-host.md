# ADR 0035: Monitoring off the host: Prometheus, Loki and blackbox to k3s, no Docker on the host

- Status: Proposed
- Date: 2026-11-21
- Deployment status: not built
- Would amend: [ADR 0019](0019-homeserver-monitoring-alerting.md) (where the stack runs; the alerting design stays)
- Tracker: home-infra-tracker#75

## Context

ADR 0019 (2026-09) built monitoring as Docker containers on the host,
the same shape as Nextcloud AIO, which was then the host's main workload,
while k3s was still a learning cluster. Since then k3s became the
production platform (Argo CD, ESO, backups, NetworkPolicies), Grafana and
Alertmanager moved there, and on 2026-11-21 Nextcloud followed
(ADR 0033); AIO is parked and goes in migration step 5.

What still runs in Docker on the host (2026-11-21):

| Container | Job | Memory |
|---|---|---|
| Prometheus | metrics, alert rules, 30 d / 2 GB (1.8 GB used) | 253 MiB |
| Loki | logs, 30 d, on the root disk | 219 MiB |
| blackbox-exporter | HTTP(S)/TLS/mail probes, certificate expiry | 43 MiB |
| Alloy | the host's journal → Loki | 71 MiB |
| node-exporter | host metrics, plus the backup-freshness textfile | 33 MiB |
| cAdvisor | metrics of the host's Docker containers | 89 MiB |

Next to them, as systemd units: the ntfy relay, the GitHub runner
exporter and the backup-freshness timer.

Prometheus on the host no longer buys independence: Alertmanager runs in
k3s, so when k3s is down no alert goes out today either. "Homeserver
unreachable" is covered from outside by the AWS check (ADR 0017).

Options considered:

- **Keep it as is.** Works; keeps a root Docker daemon on the host for
  monitoring alone once AIO is gone.
- **A separate monitoring VM.** Its own failure domain only if Alertmanager
  moves there too; costs a third VM (about 0.5–1 GiB base load) and
  another machine to maintain.
- **Prometheus, Loki and blackbox into k3s, host agents as packages.**
  Same tooling as every other app (Git, Argo CD, digest pins, Renovate,
  Trivy).

## Decision

Proposed: move what doesn't observe the host into k3s, run what does
observe the host without Docker, then remove Docker from the host.

- **Into k3s `monitoring`** (plain manifests in `k3s-apps`, like Grafana
  and Alertmanager): Prometheus, Loki, blackbox-exporter. Data on
  node-local PVs on `k3s-node-1`'s SSD, not NFS (time-series and index
  writes); the same retention as today (30 d, Prometheus 2 GB, to be
  raised once it lives on its own volume). Grafana's datasources and
  Alertmanager's source switch to in-cluster addresses.
- **On the host, without containers**: node-exporter (Debian package,
  keeps the textfile directory for backup freshness) and Alloy (Grafana's
  package) for the journal, both as systemd units from the `monitoring`
  role, listening only on `192.168.101.1` (the k3s network) for
  Prometheus in k3s. The ntfy relay, GitHub runner exporter and
  backup-freshness timer stay as they are.
- **Dropped**: cAdvisor, once AIO is removed (it would only watch the
  monitoring itself); per-Pod metrics come from kubelet already.
- **Then**: the `docker` role leaves `site.yml`, Docker leaves the host.

### Order

1. After about a week of green Nextcloud backups (not during the
   ADR 0033 observation period): build Prometheus, Loki and blackbox in
   k3s **in parallel** to the host stack, compare targets, rules and
   alerts, then switch Grafana and Alertmanager over and stop the host
   containers.
2. node-exporter and Alloy on the host as packages; avoid reboots while
   the POST halt (home-infra-tracker#41) is open.
3. With AIO's removal (ADR 0033 step 5): cAdvisor and Docker go.

## Open points for review

- **Memory on `k3s-node-1`**: 66 % used after Euro-Office (ADR 0034);
  about 0.5 GiB more brings it near 71 %. Grow the VM, or acceptable?
- **When k3s itself is down**: nothing alerts but the AWS apex check
  (unchanged from today, since Alertmanager is already in k3s). Accept,
  or add a host-side "k3s API unreachable" check through the ntfy relay?
- **Prometheus retention**: keep 2 GB, or raise it with its own PV?

## Consequences

- No container runtime on the host after AIO's removal: one root daemon
  less, a host that is quicker to rebuild.
- All of monitoring's server side is managed like the other apps.
- Monitoring depends on k3s for collection as well as alerting; a k3s
  outage blinds it (today: it blinds alerting only).
- Host metrics and logs need endpoints on the k3s network, like
  Prometheus and Loki have today in the other direction.
