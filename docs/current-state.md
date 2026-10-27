# Current Production State

What is live now. History lives in the ADRs, the git log and the archived
`home-infra-ai-context` handovers; open work is tracked in a private task
list, not here. Facts marked
"checked 2026-10-11" were read live that day (read-only SSH, `gh api`);
everything else is taken from the committed code. Revalidate before relying
on a single fact.

## Homeserver

| Item | State |
| --- | --- |
| Host | `homeserver`, `192.168.178.100` (Ansible inventory `home.yml`, user `julian`) |
| Hardware | Fujitsu ESPRIMO P900, Intel Core i5-2400 (4 cores/4 threads), no GPU |
| OS | Debian 13 (`trixie`), kernel 6.12 (checked 2026-10-11) |
| Memory | 32 GiB (checked 2026-10-11) |
| System disk | `/`, 51 GB SSD |
| Live data | `/mnt/black-hdd`, 916 GB (Nextcloud data, Deluge downloads, k3s VM disks, Paperless media) |
| Backups | `/mnt/red-hdd`, 1.8 TB: `borg/`, `aux-backups/`, `authelia-backup/`, `paperless-export/` |
| Managed by | [`home-infra`](https://github.com/KandlerLi/home-infra): `site.yml` (host) and `k3s.yml` (k3s VMs, deliberately not part of `site.yml`) |

`site.yml` roles, in order: `base`, `storage`, `authelia_backup`,
`paperless_storage`, `nfs_server`, `docker`, `deluge`, `nextcloud_aio`,
`home_agent`, `open_webui`, `monitoring`, `blocky_ipv6_relay`,
`aux_backup`. Several of them (`deluge`, `home_agent`, `open_webui`,
`authelia_backup`, `paperless_storage`) only keep host prerequisites
(service accounts, NFS-exported directories) for workloads that run in
k3s.

### Running on the host (checked 2026-10-11)

- **Nextcloud AIO** (Docker, managed by its own mastercontainer): Apache,
  Nextcloud, PostgreSQL, Redis, Imaginary, Collabora, Talk, Notify Push,
  Whiteboard. Apache listens on `127.0.0.1:11000` and on
  `192.168.101.1:11000` for k3s Traefik. The mastercontainer admin UI is
  `https://192.168.178.100:8080` (LAN only).
- **Monitoring** (Docker, `network_mode: host`): Prometheus
  (`127.0.0.1:9090`, plus `192.168.101.1:9090` for k3s Grafana),
  node_exporter, cAdvisor (`8081`), blackbox_exporter. Grafana and
  Alertmanager run in k3s (below).
- **Docker DNS**: `daemon.json` pins `dns: [192.168.178.1]` (the
  FRITZ!Box), so containers restored at boot never start without an
  upstream resolver (checked 2026-10-25).
- **systemd services**: `blocky-ipv6-relay` (`[::]:53` → Blocky in k3s),
  `home-tools` (host facts for `home_agent`, `192.168.101.1:8095`),
  `ntfy-relay` (Alertmanager → ntfy), NFS server, libvirt.
- **Timers**: `aux-backup` (Deluge config, Open WebUI data → `red-hdd`),
  `backup-freshness` (feeds the backup-age alerts).
- **Stopped leftovers, not deleted**: `shared-ingress`, `blocky`,
  `blocky-postgres` (their roles are gone; the containers stay stopped as a
  manual rollback path).

### Host shutdown order (checked 2026-10-25)

`libvirt-guests` shuts both k3s VMs down in parallel
(`PARALLEL_SHUTDOWN=2`), and `nfs-server` stops only after it
(`Before=libvirt-guests.service` drop-in), so `k3s-node-1` unmounts its
NFS volumes cleanly instead of hanging until SIGKILL.

### NFS exports to the k3s VM

All to `192.168.101.10` only, all `root_squash` (`nfs_server` asserts
both): `/mnt/black-hdd/downloads`, `/mnt/black-hdd/deluge-config`,
`/var/lib/open-webui`, `/mnt/red-hdd/authelia-backup`,
`/mnt/black-hdd/paperless-media`, `/mnt/red-hdd/paperless-export`.

## Network and public entry

The FRITZ!Box forwards TCP 80, 443, 25, 465 and 993 to the homeserver's LAN
IP (Terraform in `KandlerLi/fritzbox`, private, local apply only). `k3s_ingress_forward` DNATs those on to the k3s VM
(`192.168.101.10`): 80/443 to Traefik, 25/465/993 to Stalwart. It also
DNATs TCP+UDP 53 to Blocky, which only LAN clients reach: the FRITZ!Box
doesn't forward 53, so there is no public DNS resolver. Deluge's
BitTorrent port is a k3s `LoadBalancer` Service on the VM with no FRITZ!Box
forward (outbound peers only, enough for Deluge; decided 2026-10-12). The
k3s network `192.168.101.0/24` is reachable only from the homeserver.

Admin VPN: the FRITZ!Box's built-in WireGuard (ADR 0025,
`runbooks/fritzbox-wireguard-vpn.md`).

### Public hostnames

All routed by k3s Traefik ([`k3s-apps`](https://github.com/KandlerLi/k3s-apps), each app module's
`ingress_routes`). "Forward-auth" = Authelia, MFA-gated, before the app.

| Hostname | Service | Auth at the edge |
| --- | --- | --- |
| `jkandler.de` | 301 to `www.jkandler.de` | none |
| `www.jkandler.de` | static site, S3 + CloudFront ([`website`](https://github.com/KandlerLi/website), ADR 0016) | none (not via the homeserver) |
| `auth.jkandler.de` | Authelia | none (it is the login) |
| `nextcloud.jkandler.de` | Nextcloud AIO | none; Nextcloud's own Authelia OIDC login |
| `ai.jkandler.de` `/healthz`, `/v1/*` | `home_agent` API | forward-auth |
| `ai.jkandler.de` (rest) | Open WebUI | none; its own Authelia OIDC login, password login and signup disabled |
| `grafana.jkandler.de` | Grafana | forward-auth, then Grafana's own OIDC login |
| `home.jkandler.de` | landing page | forward-auth |
| `torrent.jkandler.de` | Deluge | forward-auth, then Deluge's own password |
| `k8s.jkandler.de` | Kubernetes Dashboard | forward-auth |
| `mail.jkandler.de` | Bulwark webmail | forward-auth; Authelia SSO into the mailbox |
| `stalwart.jkandler.de` | Stalwart | protocol paths (JMAP, CalDAV/CardDAV): none, Stalwart's own auth; `/admin`, `/account` and the management API: forward-auth |
| `docs.jkandler.de` | Paperless-ngx | none; its own Authelia OIDC login, API by token |

### DNS (Route 53)

- [`dyndns`](https://github.com/KandlerLi/dyndns): the apex `jkandler.de` A record (dynamic, updated from the
  homeserver) and CNAMEs to it for `ai`, `auth`, `docs`, `grafana`,
  `home`, `k8s`, `mail`, `nextcloud`, `stalwart`, `torrent`, `vpn`.
- [`website`](https://github.com/KandlerLi/website): `www` alias A/AAAA to CloudFront, the only AAAA record.
- [`ses-relay`](https://github.com/KandlerLi/ses-relay): MX, SPF (`-all`), DMARC and the SES DKIM records.
- Public IP values are never recorded in Git.

LAN DNS: clients use Blocky in k3s via the homeserver
(`192.168.178.100:53` over IPv4, the `blocky-ipv6-relay` over IPv6; the
homeserver's IPv6 address is pinned via EUI-64 since 2026-09-24).

## k3s cluster

Two Debian VMs on the homeserver (libvirt, `k3s_node` role), k3s installed
from a checksum-verified release binary:

| Node | Address | Size | Role |
| --- | --- | --- | --- |
| `k3s-node-1` | `192.168.101.10` | 3 vCPU, 10 GiB | control plane, all app workloads |
| `k3s-node-2` | `192.168.101.11` | 8 GiB, 100 GB disk | agent, tainted `ci=github-runner:NoSchedule`; CI runners only |

Workloads are Terraform-managed, never `kubectl apply`:

- **[`k3s-apps`](https://github.com/KandlerLi/k3s-apps)** (CI-applied on merge): `ingress` (Traefik, file
  provider only, ACME via DNS-01), `authelia`, `blocky` (+ query-log
  Postgres), `home_agent` (+ `nextcloud_tools` sidecar), `open_webui`,
  `grafana`, `alertmanager`, `node_exporter`, `deluge`, `landing_page`,
  `kubernetes_dashboard`, `sankey_export` (CronJob), `stalwart`,
  `stalwart_config` (Stalwart settings via its API), `bulwark`,
  `paperless`.
- **[`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)** (local apply only): the self-hosted
  GitHub Actions runners (one Deployment per repo, on `k3s-node-2`,
  GitHub App registration, NetworkPolicy: no inbound, outbound only DNS
  and the public internet, plus the Kubernetes and Stalwart APIs and
  Blocky/Grafana for the `k3s-apps` runner), namespaces with their Pod
  Security labels, cluster-scoped PersistentVolumes and privilege-
  defining RBAC. Per runner slot: the runner install is a `hostPath`
  seeded once per image (not per Pod), dind listens only on its Unix
  socket and retries dockerd in place, the runner registers with GitHub
  only once dind answers, and `/work` is emptied before every job.

Namespaces (tracker #32), each with its Pod Security Admission level:

| Namespace | PSA `enforce` | Holds |
| --- | --- | --- |
| `ci` | privileged | the GitHub Actions runners (dind) |
| `node-exporter` | privileged | node-exporter (host network, PID, filesystem) |
| `monitoring` | baseline (`warn: restricted`) | Grafana, Alertmanager, kube-state-metrics |
| `auth` | baseline (`warn: restricted`) | Authelia |
| `apps` | baseline (`warn: restricted`) | every other `k3s-apps` workload, Traefik included |
| `default` | baseline | no workloads; only the `prometheus-kubelet` ServiceAccount |

`k3s-apps-ci` has a Role per namespace with only the kinds that
namespace's modules create; it reaches nothing in `ci`. Volumes on
`k3s-node-1` are static `Retain` PVs under `/var/lib/k3s-apps-data`
(Blocky and Paperless Postgres, Paperless data, Bulwark, Traefik's ACME
store, Stalwart's mail); only Authelia still uses dynamic local-path
claims.

## Applications

- **home_agent**: Claude (`claude-opus-5`) via Anthropic's API for chat and
  tools; Whisper (OpenAI) for speech-to-text via its bounded relay. Tools:
  host facts (`home-tools`), Nextcloud read/write within `Shared` as the
  dedicated `home-agent` account (ADR 0021; only the second-brain share is
  editable), Shopping List items (ADR 0014). Each tool only reaches what
  Julian has shared with that account; which of them are confirmed live
  is tracked in the private task list. Open WebUI advertises only the
  synthetic model name `home-agent`. Boundaries: the
  `check-security-invariants` skill.
- **Mail**: Stalwart for `info@`/`julian@jkandler.de`; inbound on 25,
  outbound relayed through SES (eu-central-1, production access),
  IMAP 993 / submission 465 with app passwords. Settings are code in
  `modules/stalwart_config` except the authentication directory (manual,
  set to Authelia OIDC). Runbook: `runbooks/stalwart-mail-server.md`.
- **Paperless-ngx** (ADR 0023): media on `black-hdd`, nightly
  `document_exporter` to `red-hdd`, phone upload via the API with a token.
- **Sankey export** (ADR 0015): k3s CronJob, every minute, renders the
  Finanzfluss workbook to a PNG in Nextcloud via the `sankey-export`
  account.

## Backups

| What | How | Where |
| --- | --- | --- |
| Nextcloud (files + database) | AIO's built-in Borg, nightly (AIO schedule) | `/mnt/red-hdd/borg` |
| Deluge config, Open WebUI data | `aux-backup` timer, dated tarballs | `/mnt/red-hdd/aux-backups` |
| Authelia (TOTP/WebAuthn, sessions) | k3s CronJob over NFS, 14-day retention | `/mnt/red-hdd/authelia-backup` |
| Paperless | nightly `document_exporter` CronJob over NFS | `/mnt/red-hdd/paperless-export` |
| Stalwart mail | none outside the k3s VM | — |

Backup age is alerted on via `backup-freshness` + Prometheus. Restorability
is checked by `home-infra`'s `scripts/verify-backups.sh` (all checks passed
2026-10-26; `runbooks/verify-backups.md`). Everything is on the same
homeserver; there is no off-site copy.

## Monitoring and alerting

Prometheus on the homeserver scrapes the host, containers, both k3s nodes
and blackbox probes of the public hostnames (HTTPS probes also cover
certificate expiry). Alerts go to Alertmanager in k3s
(`192.168.101.10:9093`), which notifies ntfy (via `ntfy-relay`) and email
(SES). Grafana (`grafana.jkandler.de`) reads Prometheus and Blocky's
query-log Postgres; its "GitHub Runners" dashboard shows runner Pod
readiness, waiting containers, seeding, restarts and the IO load on
`k3s-node-2` and the HDDs (no GitHub-side online/busy state, no per-Pod
CPU/memory: neither is scraped). Separately, [`homeserver-health-check`](https://github.com/KandlerLi/homeserver-health-check) (Route 53
health check + CloudWatch + SNS) alerts when the homeserver is unreachable
from outside, and [`aws-budget`](https://github.com/KandlerLi/aws-budget) alerts on cost.

## AWS and GitHub control plane

- One AWS account. Day-to-day operator identity is the IAM user `julian`
  (MFA, scoped write policy + `ViewOnlyAccess`, ADR 0018); root only for
  `terraform-state` applies.
- CI reaches AWS through GitHub OIDC with a separate plan and apply role
  per repo, all defined in [`repo-infra`](https://github.com/KandlerLi/repo-infra) (local apply only, ADR
  0010). All Terraform state is in the shared S3 bucket from
  [`terraform-state`](https://github.com/KandlerLi/terraform-state).
- Secrets live in AWS Secrets Manager containers defined by
  [`secrets-manager`](https://github.com/KandlerLi/secrets-manager); values are seeded by hand and rotated per its
  `rotation-schedule.md`.
- GitHub repository settings, branch protection, Actions permissions and
  PR policy for every repo: [`repo-infra`](https://github.com/KandlerLi/repo-infra) `config.yml`. Public repos
  get branch protection (ADR 0022); `home-infra-ai-context` is archived.
  CI applies on merge with no approval step beyond the PR review.

## Known gaps

Open work is tracked privately. Notable standing gaps: no off-site backup; Stalwart
mail not backed up off the VM; the hardware watchdog did
not fire during the 2026-09-24 silent freeze; after a homeserver reboot
the public services are down for ~15 minutes and CI for ~13, because
both k3s VMs cold-start from the black HDD at once (measured
2026-10-25, HDD 94–99 % busy for 15 minutes).
