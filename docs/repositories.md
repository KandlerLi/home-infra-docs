# Workspace and Repository Map

Every repository lives under
[github.com/KandlerLi](https://github.com/KandlerLi). Local clones are
grouped by what each repo manages: AWS resources, GitHub configuration,
local-apply control-plane roots, and the homeserver/k3s workloads.

| Project | Group | Git status | Responsibility |
| --- | --- | --- | --- |
| [`home-infra`](https://github.com/KandlerLi/home-infra) | workloads | Git, `main`, public, standard branch protection (2026-09-13) | Ansible and homeserver services |
| [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) | workloads | Git, `main`, aligned with `origin/main` | Terraform (`kubernetes` provider) workloads for the personal k3s learning cluster |
| [`dyndns`](https://github.com/KandlerLi/dyndns) | AWS | Git, `main`, aligned with `origin/main` | DNS/AWS workload Terraform and protected CI/CD |
| [`website`](https://github.com/KandlerLi/website) | AWS | Git, `main`, aligned with `origin/main` | Static personal site (S3 + CloudFront) Terraform and CI/CD, deployed and live, self-hosted runner |
| [`aws-budget`](https://github.com/KandlerLi/aws-budget) | AWS | Git, `main`, aligned with `origin/main` | $10/month AWS Budget alert Terraform, self-hosted runner, applied and confirmed live |
| [`homeserver-health-check`](https://github.com/KandlerLi/homeserver-health-check) | AWS | Git, `main`, aligned with `origin/main` | Route53 health check + CloudWatch alarm + SNS email for homeserver reachability, self-hosted runner, applied and confirmed live |
| [`ses-relay`](https://github.com/KandlerLi/ses-relay) | AWS | Git, `main`, aligned with `origin/main` | SES sending identity (jkandler.de domain + Outlook recipient) relaying homeserver monitoring alert emails, self-hosted runner, applied and live (ADR 0019) |
| [`terraform-state`](https://github.com/KandlerLi/terraform-state) | control plane | Git, `main`, public, standard branch protection (2026-09-14) | Terraform root managing the central S3 backend bucket and (as of 2026-08-23) the `julian` operator IAM identity — both foundational, root-only, essentially-bootstrap-once resources (ADR 0018) |
| [`repo-infra`](https://github.com/KandlerLi/repo-infra) | GitHub | Git, `main`, public, standard branch protection (2026-09-15) -- still applied only from a trusted local controller, never via GitHub Actions | Terraform-managed GitHub repository protection and AWS OIDC/IAM deploy identities for `dyndns`/`website`/`aws-budget`/`homeserver-health-check`/`ses-relay`, plus visibility/backup-mirror support for itself and 4 other repos; applied routinely, by `julian` once its scoped policy is live |
| [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap) | control plane | Git, `main`, public, standard branch protection (2026-09-15) -- still applied only locally with the k3s node's own cluster-admin kubeconfig, never via GitHub Actions | The k3s cluster's self-hosted GitHub Actions runner (`modules/github_runner`), cluster-scoped PersistentVolumes, and privilege-defining RBAC (the Kubernetes Dashboard's own ServiceAccount, `k3s-apps`' own CI ServiceAccount) that can't live in [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) itself |
| [`gha-common`](https://github.com/KandlerLi/gha-common) | GitHub | Git, `main`, public, standard branch protection | Shared reusable GitHub Actions workflows every repo's CI calls |
| [`secrets-manager`](https://github.com/KandlerLi/secrets-manager) | AWS | Git, `main`, public, standard branch protection, CI-applied | AWS Secrets Manager containers (names only, never values) and the rotation schedule |
| [`home-infra-docs`](https://github.com/KandlerLi/home-infra-docs) | docs | Git, `main`, public, standard branch protection (2026-10-10) | The single documentation repo: current state, architecture, ADRs, runbooks |
| [`home-infra-ai-context`](https://github.com/KandlerLi/home-infra-ai-context) | docs | Archived (merged into `home-infra-docs` 2026-10) | Former cross-model context; keeps the dated handovers |

## `home-infra`

Repository: [`home-infra`](https://github.com/KandlerLi/home-infra)

Owns Ansible inventory, roles, playbooks, tests, encrypted variables, the
home-agent source, the home-tools service, Nextcloud AIO master configuration,
Traefik, the now-retired Open WebUI Docker runtime (container stopped
2026-08-30, role files still here but unreachable — the deployed runtime is
[`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' `modules/open_webui` now), deployed read-only Nextcloud tools
plus implemented-but-not-yet-deployed Nextcloud write tools (ADR 0011,
confirmation step later removed per ADR 0013 — see
`docs/runbooks/enable-nextcloud-writes.md`), implemented-but-not-yet-deployed
Shopping List item tools (ADR 0014, second Nextcloud integration reusing the
same account via the app's own OCS API — see
`docs/runbooks/enable-nextcloud-shopping-list.md`), an
audio-transcription relay (ADR 0012, implemented, live-tested, and fixed
three times against real behavior — see
`docs/runbooks/enable-openai-voice-transcription.md`), storage, Docker,
libvirt, the GitHub runner VM, and the Finanzfluss Sankey exporter
(ADR 0015, a third dedicated Nextcloud account plus this repo's first
non-containerized service — see `docs/runbooks/enable-sankey-export.md`),
and a Traefik router that redirects the bare `jkandler.de` apex to
`https://www.jkandler.de/` (`shared_ingress_apex_redirect_enabled`, ADR
0016's Update section — no DNS/AWS change, deployed and live), and (ADR
0017/0019) a `monitoring` role — Prometheus, Grafana, node_exporter,
cAdvisor, blackbox_exporter, and Alertmanager — **applied and live**,
including Grafana published externally at `grafana.jkandler.de` via
`shared_ingress` (own Traefik Basic Auth credential in front of its own
admin login, same shape as Deluge); see `monitoring`'s own paragraph
below for what it covers, the live bugs found and fixed during
deployment, and why every container runs on `network_mode: host`. Also
(ADR 0020) a `blocky` role — network-wide DNS ad-blocking, built and
unit-tested but **not yet applied** (`blocky_enabled` stays `false`,
the role's own default, until the user validates it live) — see
`blocky`'s own paragraph below. Also (starting 2026-08-28) `k3s_node`
and `nfs_server` roles supporting a personal, deliberately-separate k3s
learning cluster — see `k3s_node`'s own paragraph below and `k3s-apps`'
own project section.
Current orientation revision: `add8273`. Has checks-only CI
(`.github/workflows/checks.yml`, added 2026-08-23) — Ansible syntax
check + full unit test suite on every push to `main`, self-hosted home
runner (switched from GitHub-hosted the same day, for consistency with
`website`/`aws-budget` — no real availability tradeoff either way here,
since these checks never touch the homeserver). Deliberately never
applies anything: `home-infra`'s real target (the physical homeserver)
has no public API, unlike `dyndns`/`website`'s AWS deploys, so applying
still always means `ansible-playbook site.yml --ask-become-pass` run by
a human over the local network, same as always. Also registers
self-hosted runners for `aws-budget`, `homeserver-health-check`, and
`ses-relay` now, via `scripts/sync_github_runner_repositories.py`
reading `repo-infra`'s `config.yml`. A second, separate workflow
(`.github/workflows/build-home-agent.yml`, added 2026-08-30) builds and
pushes `home_agent`'s image to a private GHCR package
(`ghcr.io/kandlerli/home-agent`) on the same self-hosted runner, on
every push touching `ansible/roles/home_agent/files/agent/**` --
needed once `home_agent` started moving into the k3s cluster
([`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' `modules/home_agent/`), specifically to avoid
building it on the homeserver itself, unlike every other image this
repo runs (which `docker_image`/`docker build` locally on the
homeserver as part of the normal Ansible apply). Uses the workflow's
own short-lived `GITHUB_TOKEN`, not a durable credential -- `k3s-apps`'
own pull secret is the only place a durable, read-scoped PAT is needed.

### `monitoring` role

Covers ADR 0017's four scope areas: host health (node_exporter), container
health (cAdvisor — read-only `docker.sock`, precedented by
`nextcloud_aio`'s own master container already doing exactly that), service
reachability and certificate/DNS expiry in one tool (blackbox_exporter's
HTTPS probe module reports both), all scraped by Prometheus (TSDB on the
root disk, not the HDD, retention capped at 30 days/2 GB given only 23 GiB
was free at last check) and visualized in Grafana (three provisioned
dashboards, admin password via the `_FILE` env-var convention rather than
raw env, same reasoning as `home_agent`'s `OPENAI_API_KEY_FILE`).
Alertmanager routes every firing alert to both a ntfy push notification and
an email relayed through `ses-relay` (applied and live). Every container
runs `network_mode: host` uniformly.

**Applied and verified live, 2026-08-24/25.** Deployment surfaced several
real bugs, all fixed and covered by regression tests — see ADR 0019's
Update section for the full list, but the headline ones: cAdvisor's
image-baked healthcheck probed `localhost`, ambiguous under
`network_mode: host`'s shared `/etc/hosts` (resolves to both `127.0.0.1`
and `::1`; BusyBox `wget` tried IPv6 first and never fell back), reporting
unhealthy despite the service being fine; Alertmanager's own config file
was owned by root at `0600` while the container runs as its own non-root
user, so it couldn't read its own config; and blackbox_exporter had no
credential to send for `torrent.jkandler.de`/`ai.jkandler.de` (behind
Traefik Basic Auth, only the bcrypt hash is stored anywhere in this repo),
so `ServiceUnreachable` fired permanently until those two moved to a
401-tolerant probe module. Live verification confirmed: every Prometheus
target `up`, the cert-expiry metric sane for every HTTPS target, a real
test alert routed through Alertmanager to both channels, and Grafana
published externally with a valid Let's Encrypt cert. The ntfy channel
shows Alertmanager's raw JSON as the message text (no native Alertmanager
integration) — confirmed live, and deliberately left as-is: email already
carries the readable detail, and a reformatting relay wasn't judged worth
the new container/maintenance surface.

### `blocky` role -- deleted

Built as a homeserver Docker role for network-wide DNS ad-blocking
(ADR 0020), applied and confirmed working briefly there in late
August 2026 -- but this role no longer exists in this repo at all
(`ansible/roles/blocky/` is gone). Blocky moved to a k3s-native
Deployment ([`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' own `modules/blocky`) on 2026-09-01,
alongside the `shared_ingress`-to-k3s cutover below, reached from the
LAN via `k3s_ingress_forward`'s own DNAT relay rather than binding
directly to the homeserver's LAN IP. See `history.md`'s own
"k3s learning cluster" section and its 2026-09-16 correction note for
the full history, including a real production DNS outage caused by
this move's own DNAT rule sitting committed-but-unapplied for two
weeks.

### `k3s_node` and `nfs_server` roles

A personal, single-node k3s learning cluster (2026-08-28 onward),
deliberately kept separate from this repo's production services: its
own playbook (`ansible/playbooks/k3s.yml`), never imported into
`site.yml`. `k3s_node` provisions an isolated Debian VM
(`k3s-node-1`, `192.168.101.10`, its own libvirt NAT network
`192.168.101.0/24`/`virbr11`, mirroring `github_runner`'s own
`provision_vm` pattern) and installs k3s on it from a checksum-verified
release binary under a hand-templated systemd unit — never the
`get.k3s.io` install script piped through a shell. `nfs_server` exports
specific `/mnt/black-hdd` directories to specific single-IP clients
only (never a whole parent directory, a subnet, or `no_root_squash`),
added when Deluge's migration needed real shared storage with the
homeserver rather than an isolated sandbox copy.

Cluster workloads themselves live in a separate, Terraform-managed
sibling repository, [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) — see its own project section
below. **Applied and live**: `home.jkandler.de` (2026-08-29) and
`torrent.jkandler.de` (2026-08-29/30) both now serve from there instead
of a homeserver Docker container; see `history.md`'s "k3s
learning cluster" section for the full cutover history, including why
`landing_page` was deleted outright while `deluge` was reshaped down to
host prerequisites instead.

### `aux_backup` role

Opt-in (`aux_backup_enabled`, default `false`), added 2026-09-16. Daily
`tar.gz` backup, via a systemd timer, of the small host directories
Nextcloud AIO's own Borg backup was never going to cover: Deluge's
config and Open WebUI's real data (both `nfs_server` exports, see
above) -- not another Borg repository, these sources are small enough
that Borg's own dedup/compression/encryption machinery would be
disproportionate. Runs as root (reads across several different
dedicated-host-uid directories in one pass; every hardening directive
that doesn't touch DAC/capabilities is still applied) and writes to
`/mnt/red-hdd/aux-backups`, the same disk Nextcloud's own Borg backup
uses, in its own subdirectory. See the role's own README for the full
reasoning. **Built and unit-tested, not yet applied** -- matching
`deluge`/`monitoring`'s own rollout history (private validation
before persisting `aux_backup_enabled: true` in inventory); see
the private task list's "Nextcloud data has zero backup" entry for the fuller
backup-scope investigation this came out of.

`ansible/tasks/` holds task files shared across roles/playbooks by
`include_tasks` with `role_path`/`playbook_dir`-relative paths (a new,
2026-08-21 convention in this repo — previously every `include_tasks` stayed
within its own role): `deploy_socket_service.yml` (installs a Python
service, templates its systemd unit, and waits for its socket — used by
`home_agent`; originally shared with `nextcloud_tools` too, but that role
and its whole playbook/task family, including this file's own
`generate_and_validate_nextcloud_app_password.yml` sibling, were deleted
outright once `nextcloud_tools` moved to a k3s-native replacement — see
`current-state.md`).

`ansible/inventory/group_vars/all/github_runner_repositories.yml` is
**generated**, not hand-edited: `scripts/sync_github_runner_repositories.py`
reads `repo-infra`'s `config.yml` and writes it from every repository entry
with `runner: true`. Regenerate it after changing `config.yml`, review the
diff, and commit — see `repo-infra/README.md`'s "Adding a repository with a
self-hosted GitHub Actions runner" section.

Important paths (verified against the repo 2026-09-16 -- the
`nextcloud_tools`/`shared_ingress` roles and every playbook in the old
Docker-based nextcloud-tools deploy/rotate/disable and shared-ingress
publish/rollback flows are gone outright, see this file's own
`blocky`/`ansible/tasks/` notes above and `current-state.md` for why):

```text
ansible/inventory/home.yml
ansible/inventory/group_vars/all/main.yml
ansible/inventory/group_vars/all/github_runner_repositories.yml
scripts/sync_github_runner_repositories.py
tests/test_sync_github_runner_repositories.py
ansible/playbooks/site.yml
ansible/playbooks/home-agent.yml
ansible/playbooks/k3s.yml
ansible/roles/home_agent/
ansible/roles/open_webui/
ansible/roles/nextcloud_aio/
ansible/roles/k3s_ingress_forward/
tests/test_home_agent.py
tests/test_nextcloud_tools_service.py
tests/test_open_webui.py
```

Focused validation:

```bash
cd home-infra
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/ansible-playbook ansible/playbooks/site.yml --syntax-check
.venv/bin/ansible-lint ansible/roles/home_agent ansible/roles/nextcloud_tools \
  ansible/roles/open_webui ansible/roles/shared_ingress
git diff --check
```

The whole repository has known legacy lint findings in unrelated roles. Do not
misrepresent a focused production-profile pass as a clean global lint run.

## `k3s-apps`

Repository: [`k3s-apps`](https://github.com/KandlerLi/k3s-apps)

Terraform-managed workloads (the `kubernetes` provider directly against
the cluster, not plain `kubectl apply -f` — a deliberate choice to keep
one IaC tool consistent across the homelab) for the personal,
single-node k3s learning cluster `home-infra`'s `k3s_node` role
provisions. Deliberately kept separate from `home-infra`'s own
Ansible-managed production services — services are cut over to it one
at a time, each proven live first, matching the user's explicit
"not all at once" goal.

- `landing_page.tf` — ConfigMap/Deployment/Service/Ingress serving
  `home.jkandler.de`, replacing the now fully-deleted `home-infra`
  `landing_page` role. **Cut over and confirmed live** 2026-08-29 (real
  domain, real TLS, real auth, checked in the browser).
- `deluge_storage.tf`/`deluge_secret.tf`/`deluge.tf` — two PV/PVC pairs
  bound to `nfs_server`'s NFS exports, a Secret seeding Deluge's own
  `web.conf` login (hardcoded blank/no password now -- `deluge_web_password`
  had no live consumer left anywhere by the time of the 2026-09-09
  SOPS-to-Secrets-Manager cutover and was dropped entirely rather than
  migrated, see `secret.tf`'s own comments), a Deployment (with an
  init container working around a Kubernetes `subPath`/NFS
  `root_squash` incompatibility found live), a ClusterIP web Service,
  and a `LoadBalancer` peer-port Service picked up by k3s's own bundled
  ServiceLB (BitTorrent peer traffic can't be proxied through an
  Ingress). Serves `torrent.jkandler.de`. **Cut over and confirmed
  live** 2026-08-29/30, including reading Deluge's pre-existing session
  state cleanly over NFS and a real login through the deployed UI.
- `modules/home_agent/` — `home_agent` + `nextcloud_tools` as sidecars in
  one Pod, sharing a Unix socket through an `emptyDir` (same trust shape
  the Docker deployment already had). `nextcloud_tools` reaches Nextcloud
  AIO's Apache at `192.168.101.1:11000`; the one piece that couldn't move
  as a sidecar, `home_tools_service` (reports the homeserver's own real
  hardware), stays there and is reached over a new optional TCP listener
  instead. `home_agent`'s own image is a private GHCR package built by
  `home-infra`'s self-hosted CI runner, never on the homeserver itself.
  **Cut over to `ai.jkandler.de` 2026-08-30** (see `ai_ingress.tf` below).
- `modules/open_webui/` — the k3s-native copy of `home-infra`'s own role:
  same pinned image, same real data (an NFS export of
  `/var/lib/open-webui`, not started fresh), same UID:GID the Docker
  deployment already used. Migrated against the real, already-live
  accounts/chat history — the old container was stopped first (SQLite
  tolerates concurrent NFS writers far worse than Deluge's own session
  state), confirmed to read the real data correctly, and never restarted.
  **Cut over 2026-08-30.**
- `ai_ingress.tf` — the root-level resource replicating `shared_ingress`'s
  own `ai.jkandler.de` path split (`/healthz`/`/v1/chat` to `home_agent`,
  everything else to `open_webui`) as a standard Kubernetes Ingress.
- `modules/grafana/` — Phase 1 of moving `home-infra`'s `monitoring` role
  into k3s: Grafana only (Prometheus/exporters/Blocky stay on the
  homeserver permanently, reporting this physical host's own
  hardware/being LAN-facing). No PVC — provisioned entirely from files,
  confirmed with Julian first that no click-through UI state was worth
  migrating. **Cut over 2026-08-30**; `home-infra`'s `monitoring` role
  reshaped down to nothing, no host prerequisites left behind.
- `modules/alertmanager/` — Phase 2: Alertmanager, no host dependency at
  all so it moved outright. A `LoadBalancer` Service (not ClusterIP +
  Ingress, since Prometheus needs a plain `IP:port` target, not
  Host-based routing) binds to `192.168.101.10:9093`. No PVC (its own
  data is active silences/dedup log, not real history). **Cut over
  2026-08-31**; verified end-to-end (synthetic alert reaching both ntfy
  and SES) before Prometheus was ever switched over, avoiding any
  double-fire window; `home-infra`'s `monitoring` role reshaped down to
  just the ntfy relay it still runs.
- `modules/github_runner/` — moved `home-infra`'s own self-hosted CI
  runner to k3s: real production CI for 6 repositories (`home-infra`
  itself included), so this rolled out additively (new and old runners
  serving side by side, sharing GitHub's own label-based pooling) rather
  than as a cutover window. One Deployment per repository (`for_each`),
  pinned to a second, tainted k3s node (`k3s-node-2`) dedicated to CI so
  it never competes with anything else. Docker access via a privileged
  `docker:dind` sidecar per Pod, a deliberate trade-off keeping every
  repository's existing `docker build`/`run` steps unmodified. **The old
  VM decommissioned entirely 2026-08-31** once every repository proved a
  real successful run on the new runners; `home-infra`'s own
  `github_runner` role and its playbooks deleted outright.
- `modules/ingress/` — a dedicated k3s-native Traefik Deployment,
  **cut over and confirmed live** 2026-09-01, replacing `home-infra`'s
  own `shared_ingress` role for all six public hostnames. Reached from
  the internet via a new `home-infra` `k3s_ingress_forward` role
  (iptables DNAT on the homeserver). ACME uses DNS-01 against Route53
  (lego's own provider, `dyndns`'s new `traefik-acme-dns01` IAM user),
  not the original role's own TLS-ALPN-01 — see `history.md`'s
  own "k3s learning cluster" section for the full migration writeup,
  including a real path-MTU bug found and fixed in this repo's own
  `modules/github_runner/` along the way.
- State is local (`terraform.tfstate`, gitignored) — no S3 backend like
  every other Terraform repo in this workspace, since that would put
  AWS credentials on a laptop for no benefit at this cluster's current
  value; revisit if the cluster ever holds something worth protecting
  against local state loss.
- Applying from a laptop needs an SSH tunnel through the homeserver to
  the k3s VM's apiserver (`192.168.101.0/24` has no route from anywhere
  but the homeserver) plus a one-time kubeconfig pull — see the repo's
  own README for exact commands. Confirmed against the `kubernetes`
  provider's own docs that it does **not** read `KUBECONFIG`;
  `provider.tf` sets `config_path` explicitly instead.

Validation:

```bash
cd k3s-apps
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
```

## `dyndns`

Repository: [`dyndns`](https://github.com/KandlerLi/dyndns)

Owns Terraform for Route 53 stable CNAMEs, the dynamic apex A-record updater,
API Gateway, Lambda, Secrets Manager container, the Lambda runtime IAM role,
and the protected deployment workflow.

`main` (local and `origin/main`, in sync) is at `86a9f38` (PR #8 completed
the identity handoff and protected apply; PR #9 removed the completed
state-handoff scaffolding; PR #11 squash-merged the stale-path fix, the
`protect-repository.sh` trim to only `pull_request_creation_policy` and
collaborator pruning, removal of `owner-approval.yml`/its stale
`CODEOWNERS` comment, and moving CI to the self-hosted runner; PR #12-14
squash-merged the `torrent`/`grafana`/`home` subdomains; PR #15 moved
self-hosted CI to run inside a repo-defined Docker image (`checks.yml`
gained a `build-image` job pushing `ghcr.io/kandlerli/dyndns-ci`, every
other job runs inside it via `container:`); PR #16 fixed stale README
references; PR #17 dropped `terraform.tfvars`/`.example` to match
`website`/`ses-relay`; PR #18 added `traefik-acme-dns01` for
`k3s-apps`' own ACME DNS-01 challenge, plus an `action-timeout-s` fix on
the credentials step (see `history.md`'s "k3s learning cluster"
section for the path-MTU bug this PR's own checks kept hitting); PR #19
widened that user's own policy with `route53:ListResourceRecordSets`,
found missing only once Traefik made its first real DNS-01 attempt). The
root no longer owns the account-wide GitHub OIDC provider or the
repository plan/apply roles and policies. Merging now only requires the
`Validate`/`Terraform plan` checks, since only the owner can merge at all
(collaborator-restricted) and the `production` environment's reviewer gate
covers the equivalent checkpoint before AWS changes apply. `checks.yml` and
`apply.yml` both run on the self-hosted runner (`[self-hosted, home,
debian]`) — verified production-proven via PR #11's merge, including a real
`Apply saved plan` run on `github-runner-dyndns`.

`deployment.auto.tfvars` currently includes `ai`, `grafana`, `home`,
`nextcloud`, and `torrent` in the `subdomains` set. Terraform creates all
five as CNAMEs to `jkandler.de` using the shared TTL. Do not create these
manually in Route 53.

Also owns `traefik-acme-dns01` (added 2026-09-01, PR #18/#19), a
separate long-lived IAM user (not a role -- k3s's Traefik runs
on-prem, nothing to assume a role from) scoped to exactly
`route53:ChangeResourceRecordSets`/`ListResourceRecordSets`/`GetChange`
on this same hosted zone, for `k3s-apps`' own `modules/ingress/`
Traefik's ACME DNS-01 challenge. Its own deploy-role IAM grant (real
production-mirroring least-privilege, same shape as `ses-relay`'s own
`ManageSmtpUser` precedent) lives in `repo-infra`'s `aws_policies.tf`,
not here.
(GitHub squash-merges, which diverges local unsquashed history from the
squash commit).

Validation:

```bash
cd dyndns
terraform fmt -check -recursive
python3 -m unittest discover -s tests -v
terraform init -backend=false
terraform validate
```

Normal changes use a branch, pull request, speculative plan, merge, and the
protected `main` apply workflow. Never commit Terraform state or credentials.
The reusable governance, OIDC, IAM, workflow, and runtime pattern is documented
in [Secure Repository and Deployment Blueprint](repository-security-blueprint.md).

## `website`

Repository: [`website`](https://github.com/KandlerLi/website)

Owns Terraform for a static personal CV site served at
`https://www.jkandler.de/`: a private S3 bucket reached only through
CloudFront Origin Access Control, an ACM certificate in `us-east-1`
validated via Route53 DNS records, and `A`/`AAAA` alias records for
`www.jkandler.de`. Deliberately does not touch the `jkandler.de` apex,
which stays owned by `dyndns`'s home-network DynDNS record (though the
bare apex now 301-redirects to `www` via Traefik — see below) — see
`home-infra-docs/docs/adr/0016-personal-website.md` for the full
apex-vs-subdomain reasoning.

Content lives in `site/` (plain HTML/CSS, no build step, no framework;
real CV content, not a placeholder) and deploys via `aws s3 sync` plus a
CloudFront invalidation in `apply.yml`, right after `terraform apply` —
not via Terraform-managed `aws_s3_object` resources. CI runs on the
self-hosted home runner (`[self-hosted, home, debian]`), same as
`dyndns` — originally GitHub-hosted specifically to keep the deploy
pipeline independent of home-network uptime, switched on 2026-08-23 at
the user's explicit request after being shown that tradeoff directly
(see ADR 0016's Update section).

**Deployed and confirmed live** by the user: `https://www.jkandler.de/`
serves the site.

A second, unrelated static site stack for the bare `jkandler.de` apex was
found live during troubleshooting why the apex itself didn't show this
site (it never should — see ADR 0016) and has since been fully
decommissioned; see
[the archived handover](https://github.com/KandlerLi/home-infra-ai-context/blob/main/handovers/2026-08-23-decommission-legacy-whoami-site.md). It had no
connection to this repository beyond sharing the same domain name.

Validation:

```bash
cd website
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
```

## `aws-budget`

Repository: [`aws-budget`](https://github.com/KandlerLi/aws-budget)

Owns Terraform for a single `aws_budgets_budget` resource: a $10/month
cost budget with email notifications at 80% actual, 100% actual, and
100% forecasted spend. Set in USD, not EUR — this account bills in USD
(confirmed via `aws ce get-cost-and-usage`), and AWS Budgets compares
against the account's real billing currency, so an approximate EUR
conversion would just drift from round over time. `alert_email` has no
committed default (this repo is public); it's threaded through
`TF_VAR_alert_email` from an `ALERT_EMAIL` action_variables entry in
`repo-infra`'s `config.yml`, same pattern as `ROUTE53_ZONE_ID` for
`dyndns`/`website`.

Originally placed under `bootstrap/`, moved to `infra/` on 2026-08-23
after the user asked why — `bootstrap/` is specifically for
control-plane/state Terraform roots (`terraform-state`, `repo-infra`);
a budget alert doesn't manage Terraform state or identities, it's a
standing piece of AWS infrastructure like `dyndns`/`website`, just
without a deploy step beyond the one resource.

CI runs on the self-hosted home runner (`[self-hosted, home, debian]`).
This surfaced a real gap while being set up: the fine-grained GitHub PAT
used for self-hosted-runner registration (`github_runner_github_token`)
was scoped only to `dyndns`, the one repository that had ever needed it
before — it has to be manually granted access to any new
`runner: true` repository (`Administration: write`) before that repo's
runner can register. Not something Terraform/Ansible can grant on its
own, since it's tied to a personal PAT.

**Applied and pushed.** Its self-hosted runner is registered and online, and
the `aws_budgets_budget` resource itself is live — confirmed via `aws
budgets describe-budgets`/`describe-notifications-for-budget`/
`describe-subscribers-for-notification`: `monthly-cost-alert`, $10.00 USD
limit, all 3 notification thresholds (80% actual, 100% actual, 100%
forecasted), subscriber `julian.kandler@outlook.com`.

Validation:

```bash
cd aws-budget
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
```

## `homeserver-health-check`

Repository: [`homeserver-health-check`](https://github.com/KandlerLi/homeserver-health-check)

Owns Terraform for a single Route53 health check + CloudWatch alarm + SNS
email subscription, checking `https://jkandler.de/` (the bare apex, not
`www.jkandler.de`) every 30 seconds from AWS's global network and emailing
`alert_email` on state change. This is deliberately the *only* piece of
monitoring that runs in AWS — the user asked to split monitoring by cost:
most of it (host health, container health, service reachability,
certificate/DNS expiry) is planned to run entirely on the homeserver
itself for free, but homeserver-hosted monitoring can't alert about the
home network itself being unreachable, so this one small, cheap
(~$0.50-1/month), externally-placed check covers exactly that gap. The
bare apex is checked rather than `www.jkandler.de` specifically because
`www` is CloudFront/S3 (AWS-hosted, independent of the home network) —
checking it would test AWS's own reachability, not the homeserver's.
`jkandler.de` bare is the DynDNS-managed record pointing at the home IP;
getting any response from it (even Traefik's redirect to `www`) confirms
the home network, DynDNS record, and Traefik are working end-to-end.

Same public-repo pattern as `aws-budget`: `alert_email` has no committed
default, threaded through `TF_VAR_alert_email` from an `ALERT_EMAIL`
`action_variables` entry in `repo-infra`'s `config.yml`. CI runs on the
self-hosted home runner (`[self-hosted, home, debian]`).

**Applied, GitHub repo created and content pushed.** The first real
`apply.yml` run failed once: `aws_sns_topic_subscription` needs
`sns:GetSubscriptionAttributes`/`SetSubscriptionAttributes` on the
subscription's own ARN (topic ARN plus a `:<subscription-id>` suffix),
which the original `aws_policies.tf` entry didn't grant — fixed in
`repo-infra` (`7dc6468`) by widening the `ManageSnsTopic`/`ReadSnsTopic`
statements' `Resource` to cover both the topic ARN and its subscription
ARNs. **Re-run and confirmed live, 2026-08-24**: `apply.yml`'s
`GitHubActions-32659814514` run shows every step (including "Apply saved
plan") green.

## `ses-relay`

Repository: [`ses-relay`](https://github.com/KandlerLi/ses-relay)

Terraform for an SES sending identity used only to relay
`monitoring`'s (see `home-infra`'s own section) Alertmanager email alerts
to `julian.kandler@outlook.com`. Built after discovering the originally-
assumed channel — an SMTP app password on the existing Outlook address,
the same pattern `aws-budget`/`homeserver-health-check` already use for
`ALERT_EMAIL` — doesn't work: Microsoft retired app-password/basic-auth
SMTP entirely for personal Outlook.com accounts as of 2026, leaving only
OAuth2, which Alertmanager can't do. See ADR 0019.

Verifies `jkandler.de` as a sending domain (DKIM + verification via
Route53 records against the existing hosted zone, so mail goes out as
`alerts@jkandler.de`) and `julian.kandler@outlook.com` as the recipient
(SES sandbox is sufficient for one personal recipient). An IAM user
scoped only to sending from that one domain gets an access key; the
actual SMTP *password* is deliberately not computed inside Terraform
(that would put a real, usable credential in state) — a human runs
`scripts/derive_smtp_password.py` once, locally, with the raw secret key
Terraform outputs, per AWS's own published derivation algorithm.

**Applied and live.** `repo-infra`'s `config.yml`/`aws_policies.tf`
entries are applied. Getting there took three separate `repo-infra`
applies, each surfacing a real AWS constraint: `ses:VerifyDomainIdentity`/
`VerifyEmailIdentity`/`VerifyDomainDkim`/`GetIdentityVerificationAttributes`/
`GetIdentityDkimAttributes`/`DeleteIdentity` were all originally scoped to
the two specific identity ARNs, and every one was denied live regardless
of the ARN being correct — SES v1 simply has no resource-level permission
support for this action group, confirmed by trial across three separate
applies; all six now use `Resource: "*"`. Separately, `main.tf`'s DKIM
`for_each` originally iterated `aws_ses_domain_dkim.this.dkim_tokens`
directly, which fails on a first plan since that value is only known
after apply — fixed by iterating static indices instead. The domain
identity, DKIM records, and recipient identity are all verified; the SES
confirmation email to `julian.kandler@outlook.com` was clicked. The SMTP
password was derived via `scripts/derive_smtp_password.py` and written to
`home-infra`'s SOPS secrets at the time (since migrated to the
`home-infra/monitoring` AWS Secrets Manager group, the 2026-09-09
SOPS-to-Secrets-Manager cutover) — `monitoring`'s Alertmanager is sending
through it live.

Validation:

```bash
cd ses-relay
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
```

Validation:

```bash
cd homeserver-health-check
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
```

## `repo-infra`

Repository: [`repo-infra`](https://github.com/KandlerLi/repo-infra) (renamed from `github-infra` on
2026-08-21, absorbing the formerly separate `aws-account-bootstrap` root —
see `home-infra-docs/docs/adr/0010-consolidate-repo-and-identity-control-plane.md`)

Terraform root that manages, via a shared `./modules/repo` module invoked
once per repository declared in `config.yml`, GitHub-side settings and (for
AWS-deploying repos) OIDC/IAM identities for every repository in this
workspace, including itself. It now has a private `KandlerLi/repo-infra`
GitHub remote of its own (2026-08-23, disaster-recovery audit — this whole
workspace previously existed only on one laptop with zero backup) — but
this is a **backup mirror only**, pushed to for git-history durability.
Applying it still must only ever happen from a trusted local controller,
never through GitHub Actions or a role it manages; nothing about the
backup remote changes that.

- GitHub-side repository protection for `dyndns`, `website`, `aws-budget`,
  `homeserver-health-check`, and `ses-relay`: repository settings,
  `github_branch_protection` (a hardcoded baseline applied to every managed
  repository — no force-push/deletion, signed commits, linear history,
  admin enforcement, plus each repository's own
  `required_status_check_contexts` from `config.yml`), Actions
  variables, a `production` environment, the Actions allowed-actions
  allowlist plus `sha_pinning_required` (hardcoded `true` for every managed
  repository), and repository default workflow permissions
  (`github_workflow_repository_permissions`)
- The account-wide GitHub Actions OIDC provider (a single root-level
  resource) and, for repositories with an `aws_policies.tf` entry (`dyndns`,
  `website`, `aws-budget`, `homeserver-health-check`, `ses-relay`), per-repository AWS
  IAM plan/apply roles and policies, with the resulting role ARNs wired
  into `AWS_ACCOUNT_ID`/`AWS_ROLE_ARN`/`AWS_PLAN_ROLE_ARN` Actions
  variables — real Terraform references, not a guessed ARN string or a
  manually-pasted GitHub UI value

A repository entry may also set `runner: true` (`dyndns`, `website`,
`aws-budget`, and `homeserver-health-check` as of 2026-08-23 — `website`
switched from GitHub-hosted at the user's explicit request; see ADR
0016's Update section). This root never reads that key itself — it
exists purely as a convention consumed by `home-infra`'s
`scripts/sync_github_runner_repositories.py`, which regenerates that
repository's `github_runner_github_repositories` Ansible variable.
Attaching a self-hosted runner is inherently imperative (registration
token + `config.sh` against the runner VM), which Terraform's GitHub
provider has no resource for — and the fine-grained GitHub PAT used for
that registration (`github_runner_github_token`) has to be manually
re-scoped to include each new `runner: true` repository; it doesn't
happen automatically just because `config.yml` changed.

Current orientation revision: `09afb3c` (adds the `homeserver-health-check`
`config.yml`/`aws_policies.tf` entries — Terraform validated, **not yet
applied**). The prior `a55d294` (`github_branch_default` pinning every
managed repo's default branch to `main`) and the scratch `testing`
repository's removal (`9e9fc16`) are both applied; `aws-budget`'s stray
`initial-setup` branch was deleted once its default branch flipped to
`main`.

State is on the shared encrypted S3 backend key `repo-infra/terraform.tfstate`
(migrated from a local backend on 2026-08-21; the local-backend gap noted in
earlier sessions is resolved). Run this root only from a trusted local
controller with a human-selected, short-lived AWS identity plus a GitHub
token. Never add GitHub Actions for it and never use one of its managed roles
to plan or apply it. `prevent_destroy` is set on the OIDC provider and both
IAM roles.

The 5 AWS resources (OIDC provider, both IAM roles, both inline role
policies) were absorbed from `aws-account-bootstrap` on 2026-08-21 via the
same no-destroy `removed`/`import` handoff pattern used for the original
`dyndns` → `aws-account-bootstrap` migration: a `terraform plan` in
`repo-infra` showed only the 5 imports plus 2 purely cosmetic IAM role
`description` changes (trust policies were byte-identical) before applying,
`aws-account-bootstrap` then relinquished the same 5 resources, and a
post-migration `gh api` cross-check confirmed `dyndns`'s live Actions
variables still match the real role ARNs — its CI required no changes.
`aws-account-bootstrap` was then deleted (it was local-only with no remote
and only 4 commits; full history is preserved in `repo-infra`'s commit log
and the dated handover).

`whoami` has pre-existing manual GitHub protection (a disabled ruleset,
non-default merge settings, unrestricted Actions permissions) but is
**intentionally excluded** — do not add it to `config.yml` without a separate
decision, since the user may archive or delete that repository.

See `decisions.md`,
`docs/adr/0009-terraform-managed-github-repository-protection.md`,
and `home-infra-docs/docs/adr/0010-consolidate-repo-and-identity-control-plane.md`.

Validation:

```bash
cd repo-infra
terraform fmt -check -recursive
terraform validate
terraform plan
```

## `home-infra-docs`

Repository: [`home-infra-docs`](https://github.com/KandlerLi/home-infra-docs)

The single documentation repository: current state, architecture, ADRs,
decisions log, repositories, operations and runbooks. Update it in the same
change window as material architecture or production-state changes.

Key entry points:

```text
README.md
docs/current-state.md
docs/architecture/README.md
docs/networking.md
docs/security-and-secrets.md
docs/runbooks/publish-home-agent.md
docs/runbooks/publish-open-webui.md
```

## `home-infra-ai-context` (archived)

Repository: [`home-infra-ai-context`](https://github.com/KandlerLi/home-infra-ai-context)

Merged into `home-infra-docs` in 2026-10 and archived. Its living context
files now live here under `docs/`; its agent rules moved to the owner's
local agent configuration. It is kept read-only for its dated
`handovers/` (2026-08-19 to 2026-08-30).

## `terraform-state` project and S3 backend

Repository: [`terraform-state`](https://github.com/KandlerLi/terraform-state)

Git repository on `main`, with a private GitHub backup mirror (added
2026-08-23's disaster-recovery audit, like `repo-infra` and 3 others).
It manages the central S3 bucket with:

- S3 versioning enabled
- AES-256 server-side encryption
- all public access blocked
- no `force_destroy`, so a non-empty/versioned bucket resists deletion

**As of 2026-08-23** (`operator.tf`, ADR 0018), it also manages the
`julian` operator IAM identity — the day-to-day login for running
`repo-infra` locally, replacing routine AWS root use. Merged here from a
short-lived separate `operator-identity` root rather than kept standalone
or folded into `repo-infra`: both this bucket and that identity are
foundational, essentially-bootstrap-once resources applied only by root,
the same trust level and cadence, while `repo-infra` is applied routinely
by `julian` once its policy here is live. `julian`'s write policy grants
it zero access to this root's own state or to its own user/policy
resources, so this root can only ever be planned or applied as root.

**Applied and fully verified live, 2026-08-24.** `AdministratorAccess`
is confirmed detached (`aws iam list-attached-user-policies` returns
empty for the write-scoped policy's own permissions — the call itself
now needs `ViewOnlyAccess`, see below). A clean `repo-infra` `terraform
plan` (`No changes`) ran successfully under `julian`'s scoped policy
alone. Two gaps were found and fixed live during rollout: `aws login`'s
OAuth flow needed two `signin:*` actions root never required (invisible
until `AdministratorAccess` was actually gone), and the original
write-only policy couldn't do any ad-hoc read-only verification, fixed
by attaching AWS's managed `ViewOnlyAccess` policy (deliberately not the
broader `ReadOnlyAccess`, which would allow reading actual data like S3
object contents or secrets). See ADR 0018's 2026-08-24 Update for the
full story of both.

It was originally bootstrapped with local state because the bucket did not yet
exist. `backend.tf` now configures this root to use the bucket it manages:

```text
bucket: jkandler-terraform-state
region: eu-central-1
State-bootstrap key: terraform-state/terraform.tfstate
State-bootstrap lock: terraform-state/terraform.tfstate.tflock
DynDNS state key: dyndns/terraform.tfstate
DynDNS lock key: dyndns/terraform.tfstate.tflock
Repo-infra key: repo-infra/terraform.tfstate
Repo-infra lock: repo-infra/terraform.tfstate.tflock
Website state key: website/terraform.tfstate
Website lock key: website/terraform.tfstate.tflock
AWS Budget state key: aws-budget/terraform.tfstate
AWS Budget lock key: aws-budget/terraform.tfstate.tflock
Homeserver health check state key: homeserver-health-check/terraform.tfstate
Homeserver health check lock key: homeserver-health-check/terraform.tfstate.tflock
SES relay state key: ses-relay/terraform.tfstate
SES relay lock key: ses-relay/terraform.tfstate.tflock
```

The `julian` operator identity (`operator.tf`, above) lives in this same
`terraform-state/terraform.tfstate` key — it's part of this root, not a
separate one, so it needs no state key of its own.

The `aws-account-bootstrap/terraform.tfstate` key is residual from the now-
deleted `aws-account-bootstrap` root (relinquished, not actively used); its
resources were migrated into the `repo-infra` key above on 2026-08-21.

The backend enables encryption and native S3 lock files. The `dyndns` plan role
can read only its state object; the apply role has scoped read/write access to
its own state and lock. Raw state can contain sensitive values and must never be
committed, pasted into chat, or copied into either documentation repository.

Observed without reading state contents on 2026-08-19:

| Local artifact | Size | Mode | Meaning |
| --- | ---: | ---: | --- |
| `terraform.tfstate` | 0 bytes | `0664` | Empty local state placeholder |
| `terraform.tfstate.backup` | 6,571 bytes | `0664` | Non-empty historical local state backup |
| `.terraform/terraform.tfstate` | 1,580 bytes | `0664` | Local backend metadata |

All are ignored by `.gitignore`, but ignored does not mean protected. The
non-empty backup may contain infrastructure metadata or sensitive values and is
world-readable to local users because of mode `0664`. Do not inspect, delete,
move, chmod, migrate, or publish this project without explicit user
authorization and a recovery-aware plan.

Recommended future work is to verify the remote migration and recovery copy,
tighten local state-artifact permissions, publish the existing filtered Git
repository, and then apply the repository-security blueprint with
state-specific permissions.
