# History: build and migration record

**Not maintained.** This is the long-form `current-state.md` as it stood
on 2026-10-10, moved here unchanged when `current-state.md` was cut down to
the live state (2026-10-11). Code comments and runbooks point at its
sections ("k3s learning cluster", "Mail server", ...) for the reasoning
behind specific workarounds; that reasoning is still valid where the code
still does what it describes. For what is live now, read
[`current-state.md`](current-state.md); statements below such as "not yet
deployed" or "still open" are as of when they were written.


## Host and storage

| Item | State |
| --- | --- |
| Inventory host | `homeserver` |
| LAN address | `192.168.178.100` |
| OS | Debian 13.6 (`trixie`) |
| Kernel | `6.12.101+deb13-amd64` |
| Ansible Python | `/usr/bin/python3` |
| Memory | 16 GiB online; approximately 15.5 GiB usable; no offline blocks |
| Large data mount | `/mnt/black-hdd` on `/dev/sda1` |
| Failed systemd units | None observed at final validation |

The memory was upgraded from 8 GiB to 16 GiB on 2026-08-19. The fresh boot's
accessible kernel journal contained no memory, EDAC, MCE, or hardware-error
indicators. This establishes successful recognition and initial operation, but
an exhaustive offline memory test was not run.

## Public and loopback services

| Endpoint/bind | Owner | Expected result |
| --- | --- | --- |
| TCP 80/443 | iptables DNAT (`k3s_ingress_forward`) → k3s-native Traefik | Forwards straight through to `192.168.101.10:80`/`:443` at the packet level; `shared-ingress` itself retired 2026-09-01, hostname routing now happens entirely inside k3s -- see "k3s learning cluster" below |
| TCP+UDP 53 on `192.168.178.100` (the homeserver's real LAN IP) | iptables DNAT (`k3s_ingress_forward`) → k3s-native Blocky | Forwards to `192.168.101.10:53`; rule committed to the role through 2026-09-05 but not actually applied live until 2026-09-16 (the role's own playbook, `ansible/playbooks/k3s.yml`, is deliberately excluded from the routine `site.yml` apply) -- caused a real production DNS/probe outage once the LAN was first pointed at this path, see `PARKED.md`. Confirmed live now via `iptables -t nat -L PREROUTING` packet counters and the Blocky-Postgres Grafana dashboard showing real queries |
| `[::]:53` on the homeserver | `blocky-ipv6-relay.service` (systemd, `blocky_ipv6_relay` role) → k3s-native Blocky | Bridges IPv6 DNS queries to Blocky's IPv4-only k3s address (`192.168.101.10:53`) -- the homeserver's own IPv6 address isn't currently stable (SLAAC/`dynamic`, not pinned), so nothing on the LAN is actually pointed at this relay as of 2026-09-16; see `PARKED.md` |
| `127.0.0.1:8082` | Traefik ping | Retired 2026-09-01 with `shared-ingress` itself -- the new Traefik's own health entrypoint lives only inside its k3s Pod's network namespace, not the homeserver's loopback |
| `127.0.0.1:11000` | Nextcloud AIO Apache | Reverse-proxy upstream only; reached from k3s over `192.168.101.1:11000`, the homeserver's own address on the k3s VM's isolated network |
| `nextcloud.jkandler.de` | k3s-native Traefik to AIO | Public HTTP/2 200 |
| `127.0.0.1:8090` | `home-agent` (Docker, homeserver) | **Stale -- corrected 2026-09-16**: no such container exists (confirmed live, `docker ps -a` on the homeserver has no `home-agent` entry at all). `home_agent`'s own role was reshaped down to just host prerequisites the k3s Pod still depends on (`home_tools_service`, the canonical CI-build source), same shape as `deluge`'s own reshape -- there is no homeserver-side container left to bind anything |
| `ai.jkandler.de/healthz` | k3s-native Traefik (`192.168.101.10:80`, reached via DNAT) | `home-agent-api-chain` (named `agent-chain` before 2026-09-25) -- Authelia forward-auth (MFA-gated; redirects to `auth.jkandler.de` without a valid session), not Traefik Basic Auth since 2026-09-08; served by `infra/k3s-apps`' `home_agent` module, not the homeserver container -- see "k3s learning cluster" below and its own Authelia SSO entry |
| `ai.jkandler.de/v1/chat` | k3s-native Traefik (`192.168.101.10:80`, reached via DNAT) | Same auth as `/healthz`; authenticated JSON POST |
| `ai.jkandler.de/` | k3s-native Traefik (`192.168.101.10:80`, reached via DNAT) | Public login page HTTP 200 (real `<title>Open WebUI</title>` confirmed); served by `infra/k3s-apps`' `open_webui` module -- deliberately *not* on `home-agent-api-chain`'s Authelia forward-auth (its `open-webui-chain` carries no auth middleware), since Open WebUI does its own app-level Authelia OIDC login instead. Native email/password login and signup are both disabled at the protocol level since 2026-09-08 (`ENABLE_PASSWORD_AUTH`/`ENABLE_SIGNUP=False`) -- Authelia OIDC is the only way in; see `docs/home-infra-docs/docs/runbooks/harden-open-webui-login.md` for a 2026-09-11 gotcha (a pre-existing database kept rendering the now-decorative login form) |
| `127.0.0.1:8091` | Open WebUI (Docker, homeserver) | Stopped 2026-08-30 for the k3s data migration and never restarted -- superseded above |
| `/run/nextcloud-tools/nextcloud-tools.sock` | `nextcloud-tools` | Active Unix-socket API; mounted read-only into `home-agent`; health returns 200 |
| `127.0.0.1:9090` | Prometheus | Loopback only; reachable remotely via SSH tunnel. Additionally listens on `192.168.101.1:9090` (the k3s VM's isolated network) for the k3s-native Grafana below |
| `192.168.101.10:9093` | Alertmanager | k3s-native `LoadBalancer` Service, `infra/k3s-apps`' `alertmanager` module, not a homeserver container -- see "k3s learning cluster" below. Reached by Prometheus over `monitoring_alertmanager_upstream` |
| `127.0.0.1:9100` / `8081` / `9115` | node_exporter / cAdvisor / blackbox_exporter | Metrics sources scraped by Prometheus, loopback only |
| `grafana.jkandler.de` | k3s-native Traefik (`192.168.101.10:80`, reached via DNAT) | `grafana-chain` -- Authelia forward-auth gates the route (not Traefik Basic Auth since 2026-09-08); Grafana's own native login form *and* Basic Auth are both separately disabled at the protocol level, so Authelia OIDC is the only way past its own login too -- doubly gated (forward-auth, then OIDC). Served by `infra/k3s-apps`' `grafana` module, not a homeserver container -- see "k3s learning cluster" below |
| `home.jkandler.de` | k3s-native Traefik (`192.168.101.10:80`, reached via DNAT) | `home-chain` -- Authelia forward-auth (not Traefik Basic Auth since 2026-09-08); served by `infra/k3s-apps`, not a homeserver container -- see "k3s learning cluster" below |
| `torrent.jkandler.de` | k3s-native Traefik (`192.168.101.10:80`, reached via DNAT) | `deluge-chain` -- Authelia forward-auth gates the route (not Traefik Basic Auth since 2026-09-08); Deluge's own login is still a second, independent prompt behind that (no "no auth" mode to disable, confirmed against its `auth.py` source); served by `infra/k3s-apps`, not a homeserver container -- see "k3s learning cluster" below |

**Resolved 2026-08-30**: cAdvisor's port (originally `8080`, set when
`monitoring` was built, ADR 0017) silently collided with Nextcloud
AIO's own mastercontainer admin interface (`https://<homeserver>:8080`,
unrelated to Ansible, always meant to be directly reachable) -- found
only when that interface was needed again (to start the guarded
ingress-migration maintenance window below) and was unreachable.
cAdvisor moved to `8081`; nothing else used that port.

## Containers

The expected AIO set is master, Apache, Nextcloud, PostgreSQL, Redis,
Imaginary, Whiteboard, Notify Push, Talk, and Collabora. AIO manages its sibling
containers. `home-agent` and Open WebUI are managed by Ansible.

Expected critical properties:

- `shared-ingress`: **stopped 2026-09-01**, container not deleted -- moved to
  k3s (see "k3s learning cluster" below), stopped via a dedicated one-time
  playbook (`ansible/playbooks/shared-ingress-k3s-cutover.yml`, which manages
  the container directly and survives independent of the role). The
  `shared_ingress` **role itself was deleted outright the same day**,
  deliberately without the bake-in period every prior migration here
  observed -- the user explicitly chose to skip it after that tradeoff was
  raised. Its whole family of guarded publish/rollback playbooks
  (`publish-home.yml`/`publish-grafana.yml`/`publish-deluge.yml` and their
  three rollback counterparts, plus the original
  `shared-ingress-prepare.yml`/`shared-ingress.yml`/`shared-ingress-agent.yml`/
  `shared-ingress-rollback.yml`) went with it, along with `site.yml`'s own
  import and the test coverage that directly rendered its templates
  (`test_deluge.py`'s/`test_monitoring.py`'s own `DelugeIngressTests`/
  `GrafanaIngressTests` classes). `shared_ingress_enabled`/other
  `shared_ingress_*` config variables stay in committed inventory,
  harmless with the role gone; `shared_ingress_auth_password_hash` is
  still live, now consumed by `k3s-apps`' own ingress module instead.
- `home-agent`/Open WebUI: **stale -- corrected 2026-09-16, confirmed
  live neither exists as a homeserver Docker container any more**
  (`docker ps -a` has no entry for either). Both moved to k3s-native
  Deployments 2026-08-30 (`infra/k3s-apps`' `modules/home_agent`/
  `modules/open_webui`); see the "Endpoint/bind" table above and the
  "k3s learning cluster" narrative below for the current architecture.
  This bullet described the pre-migration state and was never updated
  after the cutover.
- `nextcloud-aio-apache`: healthy, `127.0.0.1:11000`
- `nextcloud-tools`: active and enabled host systemd service; service-owned
  credential and protected Unix socket, no public listener
- `prometheus`, `alertmanager`, `node-exporter`, `cadvisor`,
  `blackbox-exporter`: all healthy, all `network_mode: host`, all
  loopback-bound (Prometheus additionally on `192.168.101.1:9090` for
  the k3s-native Grafana below). Grafana itself is no longer one of
  these Docker containers -- see "k3s learning cluster" below

Open WebUI can reach the synthetic `home-agent` model over the frontend bridge;
a live outbound HTTPS check from the container was blocked.

After the guarded hardware-maintenance shutdown, the AIO application containers
remained stopped across the host boot. The user started them through the AIO
interface at `https://192.168.178.100:8080`; all expected containers and the
public Nextcloud status were healthy afterward.

## DNS

- `jkandler.de`: dynamic IPv4 A record updated by the AWS DynDNS stack.
  Traefik now has a router for this exact hostname
  (`shared_ingress_apex_redirect_enabled`) that 301-redirects to
  `https://www.jkandler.de/` with its own Let's Encrypt cert — no DNS/AWS
  change, purely a Traefik-side addition once `www` went live and the
  bare apex was found to 404.
- `nextcloud.jkandler.de`: CNAME to `jkandler.de`
- `ai.jkandler.de`: CNAME to `jkandler.de`
- `torrent.jkandler.de`: CNAME to `jkandler.de`
- `grafana.jkandler.de`: CNAME to `jkandler.de`
- `www.jkandler.de`: **live** — alias A/AAAA record to the `website` repo's
  CloudFront distribution (ADR 0016), confirmed working by the user
- no documented public AAAA records other than `www.jkandler.de` above
- transient public IP values must not be recorded in Git

A second, unrelated site stack for the bare apex was found and fully
decommissioned on 2026-08-23 — see
[the archived handover](https://github.com/KandlerLi/home-infra-ai-context/blob/main/handovers/2026-08-23-decommission-legacy-whoami-site.md). It predated
every Terraform root in this workspace (created 2024-02-12), was
unreachable since `dyndns` took over the apex A record, and had no
connection to `website`/ADR 0016 beyond sharing the same domain.

A follow-up AWS account inventory audit the same day found and
decommissioned the *rest* of that same old project — an ECS
cluster/service that had been silently failing every ~20 minutes for 2.5
years (98,075 failed attempts), an ECR repository, a custom VPC, IAM
roles, and the project's own old Terraform backend (S3 + DynamoDB). See
[the archived handover](https://github.com/KandlerLi/home-infra-ai-context/blob/main/handovers/2026-08-23-aws-account-audit.md).

**Resolved same day**: the audit's two open IAM findings. The orphaned
2021 `ecsTaskExecutionRole` (unrelated to `whoami` — predated it by two
years) was detached and deleted. The IAM user `julian`'s active,
no-MFA-protected access key (`AKIA…JJK6`, created 2026-08-17)
was deleted; only its already-inactive predecessor
(`AKIA…XSUE`, 2026-07-19) remains, untouched since it wasn't
part of what was asked. At the time, `julian` still had
`AdministratorAccess` and no MFA device — **both fully resolved
2026-08-24**, see the "AWS automation identity" section below: `julian`
now has MFA enrolled, `AdministratorAccess` replaced with a scoped
write policy plus AWS's `ViewOnlyAccess` for reads (ADR 0018).

## Desired state flags

Committed in `home-infra/ansible/inventory/group_vars/all/main.yml`:

```yaml
nextcloud_aio_reverse_proxy_enabled: true
shared_ingress_enabled: false
shared_ingress_start: true
shared_ingress_agent_enabled: true
home_agent_frontend_network_enabled: true
open_webui_enabled: true
shared_ingress_open_webui_enabled: true
nextcloud_tools_enabled: true
shared_ingress_deluge_enabled: true
monitoring_enabled: true
shared_ingress_grafana_enabled: true
```

Turning these off in an ordinary apply would be production-impacting. Use the
guarded rollback runbook instead.

`shared_ingress_enabled` is the one exception, flipped to `false` on
2026-09-01 as the immediate safety fix once `shared_ingress` itself
moved to k3s (see "k3s learning cluster" below) -- the role was
deleted outright the same day, so this flag (and the rest of this
list's `shared_ingress_*` entries) are now inert leftovers with no
role left to read them, not cleaned up as part of that deletion.

`home_agent` has no `home_agent_enabled` flag any more (removed
2026-08-28) -- the user chose to keep it permanently on rather than
opt-in, so the role is now always applied by `site.yml`, matching
`base`/`docker`/`storage`/`nextcloud_aio`. It's absent from this list
for that reason, not by omission.

`deluge_enabled` was removed from this list entirely on 2026-08-30, not
just turned off: Deluge itself no longer runs as a homeserver Docker
container (see "k3s learning cluster" below), so the `deluge` role has
no container to enable/disable any more -- its remaining tasks (service
account, NFS-exported directories, ACL) are host prerequisites the k3s
copy depends on and now always run unconditionally, the same shape as
`base`/`docker`/`storage`/`home_agent`.

This list does not yet include `blocky_enabled` -- built (ADR 0020,
2026-08-28) and now running live on the homeserver (`blocky` and
`blocky-postgres` containers confirmed up, answering real DNS queries
and the blocking-status API), but the flag itself is still set locally
(`true`) directly in the role's own `defaults/main.yml`, not committed
to inventory. Every other opt-in role's enablement flag lives in this
inventory file instead; whether to move `blocky_enabled` here to match
is an open question, asked of the user but not yet answered.

The Open WebUI flags were persisted after authenticated public login/chat
validation. The agent and Nextcloud tool flags were persisted after direct and
Open WebUI file-access validation. The running agent's configured cloud model
is `claude-opus-5` (Anthropic, `low` output effort; switched from OpenAI
`gpt-5.4-mini` 2026-09-18 and confirmed live by the user 2026-09-19), while
the frontend deliberately advertises only the synthetic name `home-agent`.

ADR 0011 (2026-08-21) adds Nextcloud write tools (create/update/delete/move)
on top of that same read-only boundary: **implemented and committed, not
yet deployed.** ADR 0013 (2026-08-22) then removed the confirmation step
that original design had — the user found the propose/confirm round trip
too much friction after using it. Writes now execute immediately through a
single `write_nextcloud_file` call (`631c8c3`); safety still comes from
scope, the extension/size allowlist, conflict-protecting conditional
WebDAV headers, the empty-folder-only delete guard, and content-free audit
logging, just not a human reviewing each write first. Deploying this needs
two things not yet done: widening the existing `home-agent` Nextcloud
share from read-only to editable (manual, same reasoning as the original
share creation — see `docs/runbooks/enable-nextcloud-writes.md`), and a
real `ansible-playbook --ask-become-pass` run plus live write/conflict
verification through an actual chat.

ADR 0014 (2026-08-22) adds a second, separate Nextcloud integration: item
tools (add/check/uncheck/remove) for the Shopping List app (`shopping_list`
by otherworld-dev), so "put milk on my shopping list" lands in that app
instead of as a text file. It reuses the same dedicated `home-agent`
account/app password via the app's own OCS JSON API, executes immediately
like ADR 0013, and needs no new Ansible role, playbook, or environment
variable — additive at the Python service/client layer only. **Implemented
and committed, not yet deployed.** The same two prerequisites as ADR 0011
apply here too: a manual step (share a Shopping List list with the
`home-agent` account, granting edit permission — see
`docs/runbooks/enable-nextcloud-shopping-list.md`) and the same
`ansible-playbook --ask-become-pass` pair, plus live end-to-end
verification (add, check off, uncheck, remove, ambiguous-name handling)
through an actual chat.

ADR 0012 (2026-08-21) fixes voice recording in Open WebUI, which was
failing with an `HF_HUB_OFFLINE`/"outgoing traffic has been disabled"
error — Open WebUI's default local-Whisper STT engine needs a Hugging
Face Hub download that this deployment's network (no egress by design)
and `OFFLINE_MODE=True` both correctly block. `home-agent` gained a new
bounded relay (`POST /v1/audio/transcriptions`, capped at OpenAI's own
25 MB limit, model always pinned server-side) and Open WebUI is pointed
at it via `AUDIO_STT_ENGINE=openai`. **Implemented and live-deployed by
the user; transcription itself now works.** Two real bugs found and fixed
via live testing: Open WebUI's aiohttp client streams the recording from
an async generator, so it sends `Transfer-Encoding: chunked` with no
`Content-Length` at all, which the relay's first version treated as a
zero-length request and rejected with `413` (fixed with chunked-body
support); and the relay only ever extracted the `file` field, silently
dropping the `language` field Open WebUI also sends, so Whisper had to
auto-detect and German speech sometimes came back as English (fixed,
`e360775`, by relaying `language` unchanged rather than pinning it like
`model`). Two navigation/UX surprises were also found and corrected
live-only (not code bugs): the engine picker is in the Admin Panel's own
Audio page, not the per-chat Settings gear (which only ever offers
Default/Web API by design and was easy to land on by mistake); and
selecting "OpenAI" there auto-fills the real `https://api.openai.com/v1`
into API Base URL, which must be hand-changed to `http://home-agent:8000/v1`
rather than being pre-filled correctly as originally assumed. See
`docs/runbooks/enable-openai-voice-transcription.md`. Final confirmation
that German transcribes correctly, after the language-relay fix is
redeployed and a personal Language setting is configured, is still
pending.

Deluge was fully deployed and published as a homeserver Docker container,
every step of `docs/runbooks/publish-deluge.md`'s phased rollout
independently verified live: `deluge_web_password` (not the default) is
confirmed to gate login via a direct JSON-API check, `occ
files_external:list` confirms the `/Downloads` mount (id `2`) is
registered against `/mnt/black-hdd/downloads` and visible only to
`admin`, and `https://torrent.jkandler.de/` was confirmed via `curl` to
return `401` unauthenticated, `200` with the correct Basic Auth
credential, `401` with a wrong one, with existing routes unaffected.
**Superseded 2026-08-29/30**: `torrent.jkandler.de` now runs from the
k3s cluster instead -- see "k3s learning cluster" below. The FRITZ!Box
port-forward for inbound peer connectivity (manual, out-of-band, not
Ansible-managed) is unaffected by that cutover and still applies.

A cold iPhone load currently fails after the logo because the deployed UI
ingress burst is 60 requests while the page references 56 same-origin assets
before startup API traffic. A controlled reproduction received HTTP 429
responses. Infrastructure revision `784ed03` raises only the UI burst to 240
and retains the 120 requests/minute sustained limit. It is deployed; a fresh
private-mode iPhone load succeeds, unauthenticated `/healthz` remains 401, and
Nextcloud returns 200.

ADR 0015 (2026-08-22) brings the previously hand-maintained
`/home/julian/sankey` Finanzfluss Sankey exporter into this repo as a new
`sankey_export` role and `sankey-export.yml` bootstrap playbook.
**Deployed by the user** (account bootstrapped, folder shared,
`sankey_export_enabled: true` set); the timer is live and firing every
minute. One live bug was found and fixed the same day: the dedicated
account's WebDAV root sees the shared folder at `Shared/Finanzen`, not
`Documents/Finanzen` — Nextcloud shares don't preserve the sharer's own
path for the recipient, confirmed via `occ share:list`. Fixed by setting
`sankey_export_remote_dir: Shared/Finanzen` in
`ansible/inventory/group_vars/all/main.yml`. A live end-to-end
confirmation (re-apply, then watch a real PNG get generated and uploaded)
is still pending. It downloads a budget spreadsheet from
Nextcloud over WebDAV, drives headless Chromium via Playwright to intercept
finanzfluss.de's own Highcharts PNG export (kept as-is; hardened against a
real HTTP 429 already), and uploads the resulting PNG back to Nextcloud —
all through a new dedicated `sankey-export` Nextcloud account (never
`admin`, which the old script used) on the loopback AIO endpoint (never the
public URL, which the old script also used). Its app password is generated
via `occ`, not pre-provisioned in a secret store -- a Nextcloud app
password can't be pre-chosen by a human, so there is nothing to put into
AWS Secrets Manager (or SOPS, before the 2026-09-09 cutover) ahead of
time; see ADR 0015 before "fixing" this. It's the first non-containerized
service in this
repo (native systemd + a Python venv). A 1-minute polling timer replaces
the old script's manual-only trigger, kept cheap by a WebDAV ETag check
that skips the expensive Playwright export unless the workbook actually
changed. Deploying this needs the same two things as every prior Nextcloud
integration: a manual share of `Documents/Finanzen` with the new account,
and a real `ansible-playbook --ask-become-pass` run
(`docs/runbooks/enable-sankey-export.md`).

ADR 0016 (2026-08-22) adds a new, separate `website` repository/Terraform
root for a static personal site at `https://www.jkandler.de/` (S3 +
CloudFront + ACM, following the same Terraform + GitHub Actions OIDC
pattern `dyndns` established). It deliberately leaves the `jkandler.de`
apex untouched — that stays `dyndns`'s DynDNS-managed record — and
deliberately uses GitHub-hosted CI runners rather than the self-hosted
home runner, so the site's own deploy pipeline doesn't depend on the home
network. **Deployed and confirmed live** by the user:
`https://www.jkandler.de/` serves the site. See
`repositories.md`'s `website` section.

## k3s learning cluster

Starting 2026-08-28, a personal, single-node k3s cluster was built as a
deliberately incremental, hands-on Kubernetes learning project -- the
user's explicit goal was "move all stuff to k3s... not migrate all
services at once, but by one." It runs in parallel with the existing
Ansible-managed homeserver, not as a replacement for it yet, and its own
playbook (`ansible/playbooks/k3s.yml`) is deliberately not imported into
`site.yml`.

- `home-infra` gained a `k3s_node` role that provisions an isolated
  Debian VM (`k3s-node-1`, `192.168.101.10`, its own libvirt NAT
  network `192.168.101.0/24`/`virbr11`, unreachable from anywhere but
  the homeserver itself) and installs k3s on it from a checksum-verified
  release binary under a hand-templated systemd unit -- never the
  `get.k3s.io` install script piped through a shell.
- `home-infra` also gained an `nfs_server` role, needed once Deluge's
  migration required real shared storage rather than an isolated
  sandbox copy: it exports specific `/mnt/black-hdd` directories to
  specific single-IP clients only (never a whole parent directory, a
  subnet, or `no_root_squash`), enforced by its own validation.
- Cluster workloads themselves are Terraform-managed (the `kubernetes`
  provider, not `kubectl apply -f`) in a new sibling repository,
  `infra/k3s-apps` -- see its own section in `repositories.md`.
- **`home.jkandler.de` cut over 2026-08-29**: the k3s-native copy
  (`infra/k3s-apps`' `landing_page.tf`) replaced `home-infra`'s
  `landing_page` role, which was then deleted outright (not just
  disabled) once the cutover was confirmed live in the browser (real
  domain, real TLS, real auth).
- **`torrent.jkandler.de` cut over 2026-08-29/30**: the k3s-native copy
  (`infra/k3s-apps`' `deluge.tf`) reads the exact same
  `downloads`/`deluge-config` directories over the new NFS export, as
  the exact same `deluge` service account (uid/gid 993/986) the
  homeserver already had. Unlike `landing_page`, the `deluge` role
  wasn't deleted -- it still owns that service account and the
  NFS-exported directories/ACL the k3s Deployment depends on, so it was
  reshaped down to just those host prerequisites (always-on, no more
  `deluge_enabled` flag) rather than removed. A real concurrent-write
  risk (both the old container and the new k3s Pod briefly running
  against the same NFS-shared session state) was caught live and
  resolved by stopping the old container before completing the routing
  cutover. A `subPath` mount for seeding `web.conf` (Deluge's own login
  config) into the config PVC hit a real `root_squash`/OCI-runtime
  incompatibility: Kubernetes creates a `subPath` mount's target file
  running as root on the node, which NFS's `root_squash` then squashes
  to an unprivileged user that can't write into `deluge-config` (owned
  `deluge:deluge`, mode `0750`) -- the Pod crash-looped with a
  permission-denied error from `runc` itself before Deluge's own
  process ever ran. Fixed with a small `init_container` that runs as
  the real Deluge uid/gid instead, doing a normal file write through
  its own mount of the same PVC.
- `shared_ingress`'s "every upstream stays on loopback" validation was
  deliberately widened exactly twice for this -- `shared_ingress_home_upstream`
  and `shared_ingress_deluge_upstream` are the only two exceptions,
  both pointing at the k3s VM's isolated-network address
  (`http://192.168.101.10:80`), never a wildcard or home-LAN address.
- **`home_agent` + `nextcloud_tools` Pod confirmed live 2026-08-30**
  (`infra/k3s-apps`' `modules/home_agent/`), cut over to
  `ai.jkandler.de` the same day (see below): `home_agent` and
  `nextcloud_tools` run as sidecars
  sharing a Unix socket through an `emptyDir`, the same trust shape
  `home-infra`'s own deployment already has. `nextcloud_tools` reaches
  Nextcloud AIO's Apache at `192.168.101.1:11000` -- this homeserver's
  own address on the k3s VM's isolated network, which required widening
  `nextcloud_aio_apache_ip_binding` and `shared_ingress_nextcloud_upstream`
  the same way (see the `nextcloud_aio`/`shared_ingress` roles' own
  comments). `home_tools_service` couldn't move as a sidecar (it
  reports the homeserver's own hardware), so it's reached over a new
  narrow, opt-in TCP listener at `192.168.101.1:8095`
  (`home_agent_tools_tcp_bind_address`) instead of its existing Unix
  socket. `home_agent`'s image is a private GHCR package
  (`ghcr.io/kandlerli/home-agent`), built and pushed by a new
  `build-home-agent.yml` workflow on the self-hosted CI runner --
  deliberately never built on the homeserver itself. Getting the Pod
  actually healthy took several real fixes past the first manifest:
  CPU limits sized without measuring actual usage starved it off the
  VM's 2 vCPU budget; a serviceaccount token neither container needs
  collided with a Secret mounted at `/run/secrets` (not
  `read_only_root_filesystem`, as first suspected -- `grafana` later
  hit the identical collision with that flag unset entirely);
  `nextcloud_tools_service.py`'s
  own loopback-only safety guard didn't know about this Pod's real
  address (widened at the source, `home-infra`'s
  `ansible/roles/nextcloud_tools/files/nextcloud_tools_service.py`, and
  re-synced into `k3s-apps`' vendored copy); the CI-built image's file
  permissions weren't world-readable (fixed with an explicit `chmod` in
  the Dockerfile, since `docker build` running as root never caught it);
  and the two sidecars' distinct UIDs didn't share a GID despite the
  Unix socket depending on one.
- **`open_webui` Pod confirmed live and `ai.jkandler.de` cut over to
  the k3s cluster, both 2026-08-30** (`infra/k3s-apps`' `modules/
  open_webui/`, `ai_ingress.tf`): unlike `home.jkandler.de`/
  `torrent.jkandler.de`'s single-service swaps, this hostname is split
  by *path* between `home_agent` and `open_webui` -- `ai_ingress.tf`
  replicates `shared_ingress`'s own `/healthz` + `/v1/chat` vs.
  everything-else split as a single Kubernetes Ingress. Migrated
  against the real, already-live data (accounts, chat history) over a
  new NFS export of `/var/lib/open-webui`, not started fresh -- the
  same choice already made for Deluge's config. SQLite tolerates
  concurrent NFS writers far worse than Deluge's own session state
  did, so unlike Deluge, the homeserver's own `open-webui` container
  had to be stopped *before* the k3s Pod could even be started, not
  just before the routing cutover -- confirmed live afterward that the
  k3s copy read the real data correctly (persisted database settings,
  e.g. `enable_signup: false`, came back instead of fresh env-var
  defaults). Rather than restoring the old container and cutting over
  separately later, the outage carried straight through into the
  `shared_ingress` routing flip in the same pass.
  `shared_ingress_agent_upstream`/`shared_ingress_open_webui_upstream`
  were widened and flipped together (they move in lockstep, since both
  branches of the path split have to reach the same k3s Traefik
  entrypoint). Surfaced one more real bug along the way: Open WebUI's
  own health check in `shared_ingress` 404'd against the new k3s
  upstream because a bare request by IP carries no `Host` header and
  Traefik there routes purely on Host -- the same fix already applied
  to Deluge's and the landing page's own checks, applied here too. The
  old `home_agent`/`open_webui`/`nextcloud_tools` Docker containers are
  still on the homeserver, just unreachable from outside now --
  reshaping those roles down to whatever host state they still own
  (mirroring `deluge`'s own precedent) is a distinct follow-up, not yet
  done. CPU is sized Burstable (100m request/500m limit), not a
  Guaranteed request copied from home-infra's own Docker
  `cpu_limit=1.0` -- that value was always a ceiling on a 4-core
  homeserver serving dozens of other containers, never a reservation,
  and copying it as a k3s request would have repeated the exact
  scheduling-budget mistake `home_agent`'s own first cut made.
- **`home_agent`/`open_webui` reshaped and `nextcloud_tools` retired
  outright, 2026-08-30** (the follow-up from the bullet above, now
  done): `home_agent` keeps only `home_tools_service` and `files/
  agent/` (the canonical CI-build source), same shape as `deluge`'s own
  reshape; `open_webui` keeps only the service account and
  `open_webui_data_dir`, its uid validated against what
  `infra/k3s-apps`' Deployment hardcodes; `nextcloud_tools` has no
  role left at all here (its only consumer, `home_agent`'s Docker
  container, was gone), though its Python source and test suite stay
  on as `infra/k3s-apps`' vendored copy's canonical original. Seven
  now-dead playbooks tied to the old Docker-based deploy/publish/
  rollback flow were deleted outright.

  **A real near-miss surfaced applying this**: the `site.yml` run for
  the `shared_ingress` Host-header fix (the bullet above, applied
  *before* this reshape existed) silently restarted the already-stopped
  `open-webui` Docker container -- `open_webui`'s role was still
  Docker-based at that point, so its own `docker_container` task
  (`state: started`) ran again as an ordinary part of that apply,
  undoing the deliberate stop from the data migration. That container
  then ran for roughly two hours, unreachable from outside (routing
  already pointed at the k3s Pod) but with an open connection to the
  same NFS-shared `webui.db` -- a real risk given the k3s Pod's own
  earlier finding that SQLite tolerates concurrent NFS writers far
  worse than Deluge's own session state does, and that this database
  is in WAL mode specifically (`-wal`/`-shm` files present), the
  journal mode most vulnerable to breaking over a network filesystem.
  Caught before any further action: `PRAGMA integrity_check` returned
  clean, real data was intact (not reset), and the stopped container's
  own logs showed no lock/corruption errors for the whole window --
  but it was pure luck that the old container saw no real write
  activity (all live traffic already went to the k3s Pod). The lesson
  this leaves for any future reshape-after-cutover step in this
  project: reshape the role (removing the container-management task)
  *before* any further `site.yml` run touches unrelated code, not
  after -- a normal `site.yml` apply for something else entirely can
  silently resurrect a deliberately-stopped container as long as its
  role still manages one.
- **Grafana cut over to the k3s cluster and `monitoring`'s role
  reshaped, both 2026-08-30** (`infra/k3s-apps`' `modules/grafana/`) --
  Phase 1 of moving the `monitoring` stack into k3s. Unlike every
  other service migrated so far, this one is a deliberate partial
  move: Prometheus, the exporters, and Blocky stay on the homeserver
  permanently, since `node_exporter`/`cAdvisor` report *this physical
  host's* own hardware/Docker daemon and `blackbox_exporter`/Blocky
  are LAN-facing -- moving them into the k3s VM would monitor the
  wrong thing entirely.
  **Superseded 2026-09-01 for Blocky specifically**: Blocky itself
  *did* move into k3s after all (`infra/k3s-apps`' own `modules/blocky`),
  alongside the `shared-ingress` cutover below -- the homeserver's
  `blocky`/`blocky-postgres` Docker containers have been stopped since
  that day (not deleted, same pattern as `shared-ingress`), and real
  LAN DNS traffic now reaches k3s-native Blocky (`192.168.101.10:53`)
  via `k3s_ingress_forward`'s own DNAT rule, same mechanism as the
  ingress cutover. "Prometheus and the exporters stay on the homeserver
  permanently" above is still accurate; only the Blocky half of that
  sentence no longer holds. This gap between decision and doc caused a
  real production DNS outage 2026-09-16 (see `PARKED.md`) once the LAN
  was actually pointed at this path for the first time -- worth
  remembering when reading any "stays on the homeserver" claim
  elsewhere in this file as of a specific date, not as a permanent
  guarantee.
  (Alertmanager was originally grouped with
  these too, but moved in Phase 2 below -- it turned out to have no
  such host dependency after all.) Only Grafana itself moved in this
  phase, both because it has no host dependency and because real
  headroom was tight (`kubectl describe node`: 350m of the VM's 2 vCPU
  budget free before this module). Its two datasources (Prometheus,
  Blocky's Postgres) needed a new path to reach it from the k3s VM: an
  **additive** widening (loopback keeps working; this only ever adds a
  second bind, never replaces the first) of Prometheus's own
  `--web.listen-address` (confirmed live it can be repeated) and
  Postgres's `listen_addresses`, both locked to `192.168.101.1` by
  their own validation -- verified reachable from inside the cluster
  (`kubectl exec` + `curl`/`nc`) before the Grafana module was even
  built. Started fresh, no PVC -- confirmed with the user first that
  nothing outside the file-provisioned dashboards/datasources was
  worth migrating (same question already asked for `open_webui`'s real
  data, answered oppositely here since Grafana's own state really is
  disposable). Two real bugs surfaced getting the Pod actually
  healthy, both the kind that don't crash-loop loudly: (1) the
  admin-password Secret mounted straight at `/run/secrets` collided
  with kubelet's own serviceaccount-token auto-mount there (same fix
  as `home_agent`'s own `automount_service_account_token = false`,
  whose own comment turned out to have blamed the wrong cause --
  corrected once this collision reproduced with no
  `read_only_root_filesystem` set at all); (2) the Secret's data key
  didn't match its own `GF_SECURITY_ADMIN_PASSWORD__FILE` path, so the
  Pod started fine but silently kept the default `admin/admin` login
  instead of the real sops password -- caught only by actually logging
  in and checking, not by the Pod's own `Running`/health-check status.
  `shared_ingress_grafana_upstream` was then widened and flipped the
  same way as every prior cutover, with the Host-header health-check
  fix (the same bug class hit live three times already: Deluge,
  `landing_page`, Open WebUI) applied proactively this time instead of
  rediscovered live a fourth time. `monitoring`'s role was reshaped
  *before* the old Grafana Docker container was touched, not after --
  directly applying the open_webui near-miss lesson above -- and
  actively removes the retired container's old config/data/secrets
  rather than leaving them to rot. Unlike `deluge`'s/`open_webui`'s own
  reshapes, this one keeps no host prerequisites at all (Grafana's
  directories were ephemeral, not real shared state), so it's purely
  subtractive; `files/dashboards/*.json` stays behind as the canonical
  source `infra/k3s-apps`' own module vendors a synced copy from. A
  third bug surfaced later: the memory limit (320Mi, matching the old
  Docker container) OOMKilled repeatedly before the liveness probe
  ever got a chance to pass -- Grafana 13's unified storage layer
  builds an in-memory bleve index for folders/dashboards/playlists on
  every startup, and a fresh Pod was already at 202Mi before that
  indexing even finished. Fixed by raising the limit to 512Mi.
- **Alertmanager cut over to the k3s cluster and `monitoring`'s role
  reshaped again, both 2026-08-31** (`infra/k3s-apps`' `modules/
  alertmanager/`) -- Phase 2 of moving the `monitoring` stack into
  k3s. Unlike Grafana, Alertmanager has no host dependency at all, so
  it moved outright rather than needing a network exception in both
  directions -- but the direction here is reversed from Grafana's own:
  Prometheus (staying on the homeserver) has to reach *this*, not the
  other way around, over a `type = "LoadBalancer"` Service
  (`modules/deluge`'s own `deluge-peer-svc` precedent -- k3s's bundled
  ServiceLB binds it directly to `192.168.101.10:9093`) rather than a
  ClusterIP + Ingress, since Alertmanager isn't browser-facing and
  Prometheus needs a plain `IP:port` target, not Host-based routing.
  Alertmanager itself keeps reaching the small stdlib ntfy-relay
  service, which stays on the homeserver, over a new additive
  `monitoring_ntfy_relay_k3s_bind_address` listener (a second
  `ThreadingHTTPServer` in a background thread, alongside the loopback
  one every other consumer on the host still uses). Prometheus's own
  `alerting.alertmanagers` target list notifies *every* address in it
  for the same firing alert (unlike a scrape target list), so unlike
  Grafana's additive datasource widenings, `monitoring_alertmanager_upstream`
  is a clean **switch** -- listing both the homeserver's own
  Alertmanager and the k3s-native one simultaneously would have
  double-fired every real notification. Started fresh, no PVC --
  confirmed with the user first that Alertmanager's own `/alertmanager`
  data (active silences, a notification-dedup log) isn't worth
  migrating. Resourced off real `docker stats` (0.77% of one core,
  17.67MiB/128MiB memory) rather than its Docker `cpus:` ceiling, the
  smallest request yet (`20m`/`100m` CPU) given only 250m of headroom
  was free before this module. Verified end-to-end before the switch,
  not after: a synthetic test alert posted straight to the k3s
  Alertmanager's own API reached both ntfy and the real SES inbox,
  *then* Prometheus was pointed at it, confirmed via its own
  `/api/v1/alertmanagers` endpoint showing exactly that target,
  healthy. `monitoring`'s role was reshaped the same way as Grafana's
  own -- landed before the old Alertmanager Docker container was
  touched, not after, directly applying the open_webui near-miss
  lesson above -- and keeps only the ntfy relay it still runs; no host
  prerequisites survive for Alertmanager itself, same as Grafana.
- **`github_runner` migrated to k3s and the old VM decommissioned,
  2026-08-31** (`bootstrap/k3s-bootstrap`'s own `modules/github_runner/`) -- a
  much bigger arc than every migration above: this is real production
  CI for 6 repositories, `home-infra` itself included, so the old
  VM-based role had to keep serving throughout rather than a cutover
  window. A **second, tainted k3s node** was added specifically for
  this -- `ansible/roles/k3s_node` gained real multi-node support for
  the first time (`k3s-node-2`, joined as an agent to `k3s-node-1`'s
  control plane, tainted `ci=github-runner:NoSchedule` so CI workloads
  never compete with or endanger anything else in the cluster), rather
  than folding CI onto the already-tight `k3s-node-1` (88% of its own
  2 vCPU budget committed before this). One Deployment per repository
  (`for_each`, the first use of that pattern in `k3s-apps`) runs the
  community `myoung34/github-runner` image with a privileged
  `docker:dind` sidecar (Docker access, the same deliberate trade-off
  every repository's existing `docker build`/`docker run` steps
  needed unmodified) -- reached over a shared Unix socket volume, not
  TCP (confirmed live that `docker:dind` never actually opens a TCP
  listener the way an early assumption expected). Registration was
  additive throughout (GitHub Actions runners sharing labels just pool
  capacity), so the new runners registered and served real jobs
  alongside the old VM's own runners with zero interruption risk,
  confirmed live with a real successful `Checks` run on every
  repository -- including a genuinely tricky one: `home-infra`'s own
  `container:` job steps first failed with a missing-Node.js bind-mount
  error (`actions/runner` bind-mounts its own `externals`/work
  directories into job containers it starts, and Docker resolves a
  bind mount's source path against the *daemon's* filesystem, not the
  client's -- with the daemon a separate `dind` container, those paths
  simply didn't exist there until shared as real volumes at identical
  paths in both containers), then a cross-machine uid mismatch (a
  workflow's own jobs splitting across the old VM and the new runner
  within one run, since the captured `id -u`/`id -g` from one machine
  means nothing on the other -- fixed at the actual source with
  `umask 000` on the runner's own process tree, after a first attempt
  using a continuously-polling `chmod -R 0777` sidecar container
  turned out to actively fight `home-infra`'s own `checks.yml` security
  step and had to be removed). Once every repository was proven, the
  old VM's runners were deregistered from GitHub directly, the VM and
  its isolated `192.168.100.0/24` network stopped and undefined via a
  one-time Ansible teardown playbook (matching this project's own
  IaC-only discipline -- not a raw `virsh` command run by hand), and
  `ansible/roles/github_runner` deleted outright from `home-infra`
  (not reshaped -- nothing on the homeserver depended on it for host
  prerequisites once the VM itself was gone), along with `site.yml`'s
  own `import_playbook` for it -- unlike every k3s-native service
  before it, this one *was* wired into the normal production apply,
  not a learning-project side project.
  - **Runner registration moved from a durable classic PAT to GitHub
    App auth, 2026-09-16** (`bootstrap/k3s-bootstrap`'s own
    `modules/github_runner`) -- the entrypoint's own `app_token.sh`
    signs a JWT with the App's private key and mints a fresh,
    short-lived installation access token at every container start,
    instead of a long-lived `ACCESS_TOKEN` secret needing manual
    rotation. Confirmed this is a genuinely separate case from GHCR
    image pulls, which stay on a classic PAT for a real platform gap
    -- see decisions.md's own "GHCR image pulls stay on a classic PAT"
    entry.
  - **A persistent per-slot Docker layer cache and Terraform provider
    cache** (`host_path`, not `emptyDir`) were added once `EPHEMERAL`
    mode was confirmed to mean a fresh Pod -- and a fresh `emptyDir`
    wiped clean -- on every single job, making `terraform init` alone
    take 5-8 minutes on every PR check, not just the first. Fixed
    since every repo's Deployment is already pinned to `k3s-node-2`
    permanently, so a `host_path` persists across Pod restarts at no
    new node-coupling cost. That persistent cache then needed its own
    bound: nothing else ever removes an image or build layer from it,
    and `k3s-node-2`'s 42GB root disk climbed 39% to 92% in ~3 days
    once every repository had its own cache per slot -- fixed with a
    small unprivileged `pruner` sidecar per Pod, running `docker image
    prune --all --filter until=24h` and `docker builder prune
    --keep-storage 2GB` on an interval, touching only images/layers
    genuinely unused so an in-flight job's own cache stays untouched.
  - **The shared runner container's own memory/CPU limits were raised
    five separate times**, each confirmed live against a real
    OOMKill or cgroup-throttling event as `infra/k3s-apps`' own
    Terraform state and provider set grew (128Mi→512Mi→1Gi memory;
    200m→1000m→2000m CPU, the node's full 2 physical cores) -- since
    this `for_each`'s single resource block covers every repository's
    own runner, each bump benefits all of them, not just whichever
    repo's job happened to trigger it. `kubectl top node` confirmed
    real headroom before each bump; CPU requests stayed fixed at
    `10m` throughout, so only the burst ceiling widened, never what
    scheduling reserves.
- **Blocky cut over to k3s, 2026-09-01** (`infra/k3s-apps`' own
  `modules/blocky/`) -- superseding its earlier "stays on the
  homeserver" plan, made viable by the same `k3s_ingress_forward` DNAT
  relay `modules/ingress` uses, extended to carry UDP+TCP DNS traffic
  too. Blocky and its query-log Postgres share one Pod (the most
  faithful translation of the Docker deployment's own `network_mode:
  host`). Several real bugs found live building it: the default
  RollingUpdate strategy briefly ran two Pods' Postgres sidecars
  against the same local-path-backed data directory at once (both land
  on the same node, and local-path has no real CSI attach/detach
  exclusivity), corrupting the WAL and losing the query-log table more
  than once -- fixed with `strategy { type = "Recreate" }`. Postgres
  needed a native sidecar (`restart_policy = "Always"` on an
  `init_container`, KEP-753), not a regular container -- as two
  ordinary containers with no ordering guarantee, Blocky attempted its
  own Postgres connection before Postgres's one-time `initdb` finished,
  exhausted its fixed 3-attempt retry budget, and permanently fell back
  to console-only logging for that Pod's whole lifetime (Blocky never
  retries the writer again). Postgres also needed `listen_addresses:
  0.0.0.0`, not the entrypoint-bypassing default -- a Kubernetes
  Service DNATs to the Pod's real IP, which nothing outside the Pod
  can reach if Postgres only listens on loopback (Grafana got a flat
  "connection refused"); `args`, not `command`, is what lets the
  official image's own entrypoint (which needs to run as root to chown
  the data directory, then drop privileges) stay in charge while still
  passing the flag through. The Service uses
  `external_traffic_policy = "Local"` -- the default `Cluster` policy
  masquerades the real LAN client's source IP before Blocky ever sees
  it, defeating the query log's whole purpose; `Local` is safe here
  since Blocky's Pod and the Service's LoadBalancer IP are both pinned
  to the same node. A `postgres-exporter` sidecar was added 2026-09-16
  after an ~18h silent query-log gap (Blocky's own Postgres writer has
  no reconnect logic, and its `up` metric stays green throughout) went
  unnoticed -- exposes `pg_stat_user_tables` metrics so Prometheus can
  alert on "no new rows in N minutes" instead.
- **`shared_ingress` cut over to k3s, 2026-09-01** (`infra/k3s-apps`'
  own `modules/ingress/`) -- the last and highest-stakes piece of the
  whole "move things to k3s, one at a time" project. Unlike every
  service before it, this one had no safe additive window: `shared_
  ingress` bound the homeserver's own physical ports 80/443 directly
  (`network_mode: host`), so only one thing could ever hold them at
  once -- a real all-or-nothing blast radius across all six public
  hostnames, accepted deliberately after being flagged clearly.
  Three decisions shaped the design: **iptables DNAT** on the
  homeserver (a new `k3s_ingress_forward` role, opt-in, off by
  default) rather than bridging the k3s VM onto the LAN -- a thin,
  invisible packet relay below the TLS layer, keeping the VM's network
  exactly as isolated as before except for this one forwarded path;
  a **new, dedicated Traefik Deployment**, not a reconfiguration of
  k3s's own bundled instance (which gets `--disable traefik` on
  `k3s-node-1`'s own systemd unit once the new one proved itself,
  `k3s_node`'s one deliberate departure from "everything else stays
  at k3s defaults"); and **DNS-01 over the original role's own
  TLS-ALPN-01** for ACME -- TLS-ALPN-01 always validates against the
  domain's real port 443, tying cert issuance to the exact moment
  80/443 got cut over with no way to rehearse it safely first, while
  DNS-01 (via lego's route53 provider and a new, narrowly-scoped IAM
  user, `dyndns`'s own `traefik-acme-dns01`) let real production
  certs be issued and proven for all six hostnames well before any
  port was ever touched.

  Two real bugs surfaced building the new Traefik itself, both fixed
  live before the first successful apply: `kubernetes_service_v1`'s
  own default wait-for-LoadBalancer-IP behavior hung indefinitely
  against k3s's still-running bundled Traefik holding the same ports
  (`wait_for_load_balancer = false`), and `kubernetes_persistent_
  volume_claim_v1`'s own default wait-for-Bound behavior hung on a
  genuine chicken-and-egg with k3s's `local-path` storage class's
  `WaitForFirstConsumer` binding mode (`wait_until_bound = false`).

  A separate, unrelated CI bug was found and fixed along the way: the
  ACME DNS-01 credential's own PR kept failing its "Terraform plan"
  check with a ~15-minute hang, 100% reproducible. Direct tcpdump
  capture during a live stall caught the real cause -- a path-MTU
  black hole, not flakiness: the k3s Pod's own flannel/VXLAN interface
  correctly sits at MTU 1450, but the self-hosted CI runners' own
  Docker-in-Docker sidecar creates a fresh custom network per job that
  stayed at Docker's untouched default of 1500, silently dropping any
  response body over ~1450 bytes crossing between them. Fixed in
  `bootstrap/k3s-bootstrap`'s own `modules/github_runner/` (`daemon.json`'s own
  `default-network-opts`, not just its plainer `mtu` key, which a
  first attempt confirmed live only covers Docker's *default* bridge
  network, never the custom ones every `container:` job actually
  runs on) -- applies to all six of this project's self-hosted CI
  runners, not just `dyndns`'s. Getting the DNS-01 IAM permissions
  fully right took two live iterations too: the deploy role initially
  lacked `iam:CreateUser` entirely (a deliberate boundary in
  `repo-infra`'s own least-privilege design, widened the same way
  `ses-relay`'s own precedent already was), and the new user's own
  policy was missing `route53:ListResourceRecordSets` (lego's route53
  provider reads existing records before writing the challenge TXT
  record, confirmed against lego's own source that `AWS_HOSTED_ZONE_ID`
  being set does still correctly skip the separate, unneeded
  `ListHostedZonesByName` call).

  The real cutover itself was sequenced for safety rather than
  following the original plan's own literal step order: DNAT was
  enabled first while `shared_ingress` stayed running (once live, all
  new external connections redirect to k3s before ever reaching
  `shared_ingress`'s own socket, so the two don't conflict, and
  disabling DNAT alone would have been an instant full rollback),
  verified end-to-end from the real outside world (`curl`/`openssl
  s_client` against every hostname, confirming both correct routing
  behavior and the genuine, freshly-issued Let's Encrypt certificate),
  and only then was `shared_ingress`'s own container stopped --
  through a new dedicated `shared-ingress-k3s-cutover.yml` playbook,
  not a raw `docker stop`, since `shared_ingress_enabled` is a pure
  `when:`-based skip-guard on the whole role, not a real toggle, and
  wouldn't have actively stopped an already-running container.
  `shared_ingress_enabled` was then flipped to `false` in committed
  inventory the same day -- a real, live risk caught applying this
  same lesson `open_webui`'s own reshape already taught: the very
  next unrelated `site.yml` apply would otherwise have silently
  resurrected the deliberately-stopped container.

  **The role itself was then deleted outright, the same day** --
  deliberately without the bake-in period every prior migration here
  observed (`deluge`, `open_webui`, `grafana`, `alertmanager` all kept
  their old role/host state around for a real proven-stable window
  first). Raised explicitly before proceeding: the user was told what
  deleting now would cost (the canonical historical config as a
  reference, the validation logic/comments explaining *why* it was
  built that way, and a fast path back to "run the old thing again" if
  ever needed later, not just "restart the already-stopped container,"
  which stays available regardless via `shared-ingress-k3s-cutover.yml`)
  -- the user chose to proceed anyway. The whole family of playbooks
  built around the role's own lifecycle went with it (nine total: the
  three guarded per-service publish playbooks and their three rollback
  counterparts, plus the original `shared-ingress-prepare.yml`/
  `shared-ingress.yml`/`shared-ingress-agent.yml`/
  `shared-ingress-rollback.yml` that first brought services onto it),
  along with `site.yml`'s own import. Test fallout went beyond the
  role's own dedicated test file: `test_deluge.py`'s and
  `test_monitoring.py`'s own `DelugeIngressTests`/`GrafanaIngressTests`
  classes directly rendered `shared_ingress`'s own `dynamic.yml.j2` to
  verify Deluge's/Grafana's own config translated correctly into it --
  no longer meaningful once nothing routes through it -- and
  `test_monitoring.py`'s own site.yml-ordering test had nothing left
  in site.yml to assert an ordering against. Full suite confirmed green
  after: 287 tests. `shared-ingress-k3s-cutover.yml` itself, and the
  already-stopped container, were deliberately left alone -- that
  playbook never invoked the role (manages the Docker container
  directly by name), so it and the rollback path it provides keep
  working unaffected by the role's own removal.
- **`sankey_export` cut over to a k3s CronJob, confirmed live
  2026-09-07** (`infra/k3s-apps`' `modules/sankey_export/`): renders
  the Finanzfluss budget Sankey diagram and uploads it to Nextcloud on
  the same 1-minute schedule the homeserver's own systemd timer used.
  `finanzfluss_export.py` needed zero code changes -- already entirely
  env-var-configured -- but the move surfaced three real bugs before
  the first genuine success: a `resources.limits`-only CPU request
  defaulted to Guaranteed QoS and reserved a full vCPU the VM's ~120m
  of free headroom couldn't satisfy (same class of bug `home_agent`'s
  own Deployment already hit); a `DirectoryOrCreate` `hostPath` for the
  ETag-cache state is created root-owned `0755` by kubelet, which
  `fs_group` does *not* fix for `hostPath` volumes specifically
  (confirmed directly via a debug Pod: still `root:root` after trying
  it) -- fixed instead with an `initContainer` running as root
  (`CAP_CHOWN` only) that chowns the mount before the real container
  starts; and `SANKEY_EXPORT_REMOTE_DIR` was copied from the role's own
  `defaults/main.yml` (`Documents/Finanzen`) rather than home-infra's
  actual live override (`Shared/Finanzen`, in `group_vars/all/main.yml`
  -- Nextcloud shares don't preserve the sharer's own path for the
  recipient), so the k3s copy 404'd on every tick while the still-live
  systemd copy kept succeeding on the correct path the whole time.
  Confirmed genuinely live via the rendered PNG's own Nextcloud
  `getlastmodified` timestamp, not just a completed Job -- only then
  was the homeserver's own `sankey-export.timer`/`.service` stopped and
  disabled, and the whole `home-infra` role deleted outright (matching
  `landing_page`'s own precedent -- it owned no host prerequisite
  anything else depended on), along with
  `generate_and_validate_nextcloud_app_password.yml`, whose only
  remaining caller this was.
- **Authelia SSO stood up and cut over, 2026-09-08 onward**
  (`infra/k3s-apps`' `modules/authelia/`, `modules/ingress/
  configmap.tf`) -- replaces Traefik's own `shared-auth` Basic Auth as
  the primary gate on five hostname chains, and adds a genuine
  MFA-gated login where the old Basic Auth had none. Authelia chosen
  over Authentik specifically because it needs no second stateful
  sidecar (SQLite is enough for a single-user homelab; the node had
  already hit real CPU/memory pressure before, see `modules/blocky`'s
  own reasoning for the same tradeoff). Stood up additively first at
  `auth.jkandler.de` (bypass-listed in Authelia's own
  `access_control`, or nobody could ever log in) and confirmed live
  with a real login + TOTP enrollment before any hostname was actually
  cut over.
  - **`agent-chain`/`deluge-chain`/`grafana-chain`/`home-chain`/
    `kubernetes-dashboard-chain`** now all carry `authelia-forward-auth`
    (Traefik `forwardAuth` against Authelia's own
    `/api/authz/forward-auth`) instead of `shared-auth`'s `basicAuth` --
    `shared-auth` itself stays defined in `modules/ingress/configmap.tf`,
    unreferenced by any chain, as a one-chain-swap rollback path.
  - **`open-webui-chain` and `nextcloud-chain` deliberately do not**
    carry `authelia-forward-auth` -- Open WebUI and Nextcloud instead
    each run their own app-level Authelia OIDC login (their own
    `authelia_oidc_*_client_secret` pair, see
    `aws/secrets-manager`), so gating the Traefik route too would
    only add a redundant, confusing extra prompt in front of the same
    MFA check. Grafana runs *both* -- `grafana-chain`'s forward-auth
    plus its own OIDC login -- a real, accepted double-gate, not an
    oversight.
  - Grafana's and Open WebUI's own native login forms/Basic Auth are
    additionally disabled at the protocol level once their OIDC
    integration was confirmed working, closing what would otherwise be
    a second, MFA-skipping credential path behind Authelia's own gate
    (Grafana: `GF_AUTH_DISABLE_LOGIN_FORM`/`GF_AUTH_BASIC_ENABLED=false`;
    Open WebUI: `ENABLE_PASSWORD_AUTH`/`ENABLE_SIGNUP=False`, plus a
    2026-09-11 fix for a pre-existing database that kept rendering the
    now-decorative login form regardless -- see
    `docs/home-infra-docs/docs/runbooks/harden-open-webui-login.md`).
  - Two container-level bugs found live right after the cutover, both
    fixed in `modules/authelia/main.tf`: the memory limit (sized against
    the portal alone, effectively single-request manual testing) was
    hit almost immediately once real traffic multiplied -- every newly
    gated service's own background polling (Grafana's `/api/live/ws`,
    Kubernetes Dashboard's refresh, etc.) calls `/api/authz/forward-auth`
    continuously, not just on a page load -- fixed by raising the
    memory limit to 512Mi/request 128Mi. Separately,
    `read_only_root_filesystem = true` made the container crash-loop
    with only a generic "Errors occurred performing startup checks"
    fatal and no further detail (a scratch-based image with no
    debugging tools); isolated via a throwaway debug Pod with identical
    volumes to a startup write somewhere outside `/data`/`/config`/
    `/tmp` -- left `read_only_root_filesystem` unset (defaults to
    `false`) rather than tracked down further, unlike every other
    container in this repo, which run fully read-only.
  - The initial forward-auth cutover of all five chains was **rolled
    back the same day** after Authelia's own SQLite storage lost its
    data twice within a few hours (a real login/TOTP validation
    succeeded, then failed to persist -- seen by the client as a bare
    login failure). Re-cut over 2026-09-08 after real load-testing
    (sequential and concurrent failed logins, a triggered regulation
    ban) failed to reproduce it, and it has not recurred through
    several more days of varied production traffic since (OIDC
    role-sync work, Nextcloud SSO's own account migration, ordinary
    daily use of all five gated services). No confirmed root cause.
    Closed as a tracked risk 2026-09-16: no recurrence since, and the
    nightly `authelia-backup` CronJob (restore-verified) means a
    recurrence no longer loses TOTP/WebAuthn registrations.
  - Per-client OIDC integration gotchas found live, each fixed in
    `modules/authelia/secret.tf`: Grafana never actually falls through
    to the UserInfo endpoint for `role_attribute_path` despite being
    documented to (a confirmed, open Grafana bug,
    grafana/grafana#106686) -- `role_attribute_path` only evaluates
    against the ID Token, so `groups` from UserInfo were silently
    ignored and every login landed as Viewer; fixed with a
    `claims_policies.groups_in_id_token` block putting `groups` back
    into the ID Token directly, applied to every client since it's
    harmless even where not confirmed necessary. Bulwark's and
    Nextcloud's clients need `token_endpoint_auth_method:
    'client_secret_post'` (client secret in the POST body) where
    Grafana/Open WebUI use the default `client_secret_basic`
    (Authorization header) -- Authelia's own token endpoint rejects a
    mismatched client outright. Bulwark's `redirect_uris` needed two
    entries (`/de/auth/callback`, `/en/auth/callback`), not the generic
    NextAuth-style path first guessed -- its frontend builds the
    callback as `$${origin}/$${locale}/auth/callback`, locale-prefixed,
    and Authelia requires an exact match per entry with no wildcards.
    Separately, the OIDC issuer's own RSA private key is interpolated
    into the YAML as a double-quoted, single-line scalar with escaped
    `\n` sequences rather than a YAML block scalar -- a block scalar
    plus Terraform's own heredoc dedent produced inconsistent per-line
    indentation that corrupted the key when parsed. On Grafana's own
    side (`modules/grafana/secret.tf`),
    `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET__FILE` (the same `_FILE`
    convention that works for the admin password) silently never got
    read -- Grafana's env-var config loader has a real gap for this
    specific nested setting (`auth.generic_oauth` already has an
    internal underscore, colliding with how `GF_<section>_<key>` parses
    section names), a documented community finding for this exact
    Grafana+Authelia combination (authelia/authelia#7203). Grafana kept
    sending `token_endpoint_auth_method: none` at Authelia's token
    endpoint regardless of what was actually mounted/set, until fixed
    with a real `grafana.ini` file instead of an env var.
  - A second Authelia identity ("admin", alongside "julian") exists
    purely so Stalwart's own admin account can SSO through Authelia
    too -- Stalwart's OIDC directory needs the resolved identity to
    match an existing Stalwart account by name exactly, and Stalwart's
    admin and Julian's own mailbox are different accounts. See the
    "Mail server" section below for the fuller Stalwart/Bulwark
    picture.
  - Every AWS Secrets Manager container (including the OIDC
    client-secret pairs -- Authelia's own pbkdf2 hash half, each
    client's plaintext half) has migrated from `bootstrap/terraform-state`
    into a dedicated `aws/secrets-manager` repo -- `home-infra/authelia`
    was deliberately last (highest blast radius, 9 keys), migrated
    2026-09-12; `bootstrap/terraform-state/secrets_manager.tf` itself is
    deleted. `aws/secrets-manager` moved from `bootstrap/` to `aws/`
    2026-09-13 once it gained real CI automation; see `decisions.md`
    for the full history.
- **`k3s-apps` ponytail-audit dedup, 2026-09-22** (`k3s-apps#83`): three
  small collapses, all `moved`-block state-preserving (confirmed via a
  clean CI plan -- 0 to add, 0 to destroy, no resource recreated).
  `modules/pvc_local_path` and `modules/ghcr_pull_secret` are new shared
  local modules replacing byte-identical `kubernetes_persistent_
  volume_claim_v1`/`kubernetes_secret_v1` resources that had been
  hand-copied across `authelia`/`blocky`/`bulwark`/`ingress` (PVCs) and
  `home_agent`/`sankey_export` (GHCR pull secrets). `modules/ingress/
  configmap.tf`'s `deluge-chain`/`grafana-chain`/`home-chain`/
  `kubernetes-dashboard-chain` Traefik middlewares -- previously
  hand-copied YAML, identical except for the name prefix and two
  numbers -- are now generated from a `locals` map via `yamlencode()`
  into a second ConfigMap key (`generated-middlewares.yml`), merged in
  by Traefik's own file provider from the same watched directory
  (`providers.file.directory`); every other middleware in that file has
  its own distinct shape and stays hand-written.
- **Traefik routes moved into each app module, 2026-09-25**
  (`k3s-apps#88`, confirmed live the same day): every app module now
  has an `ingress_routes` output (host rule, backend URL from its own
  Service, forward-auth/rate-limit/body-size settings), and
  `modules/ingress/configmap.tf` builds the whole file-provider config
  from them with one chain pattern. This supersedes the
  `generated-middlewares.yml` split above; there is a single
  `routes.yml` key again. Only Nextcloud, the apex redirect and
  `authelia-forward-auth` stay in `modules/ingress`. Old and new
  rendered config were identical router by router; internal names
  changed (`agent-chain` -> `home-agent-api-chain`, `authelia-chain`
  -> `auth-chain`; service names now equal router names). Checked after
  apply: every public hostname answers as before (forward-auth
  redirects, apex redirect, JMAP, security headers). Traefik still uses
  the file provider only and has no ServiceAccount token -- IngressRoute
  or Ingress objects were rejected because Traefik v3.7.1's Kubernetes
  providers watch every Secret in their namespaces unconditionally.
- **GitHub runner Pods isolated by NetworkPolicy, 2026-09-25**
  (`bootstrap/k3s-bootstrap#15`, `modules/github_runner/
  network_policy.tf`): no inbound; outbound only CoreDNS and the public
  internet (RFC 1918, CGNAT and link-local excluded), plus the
  Kubernetes API and Stalwart's management API for the `k3s-apps`
  runners only. Confirmed both ways: CI still passes (a full
  `k3s-apps` apply, a `home-infra` `docker build`), and a throwaway
  probe workflow (`home-infra#64`, closed) found Authelia, Stalwart,
  the Kubernetes API, the homeserver and the FRITZ!Box unreachable from
  a runner and from a container inside dind, with github.com reachable.
  Added after the same day's finding that 13 of 14 repos let anyone
  open PRs (fixed via `repo-infra`'s `pr_policy.tf`, see
  `decisions.md`).
- **k3s-node-2 disk incident closed, 2026-09-25**: the node's disk is
  98 GiB since `home-infra#52`, and free space held between 51 and 62
  GiB over five days of CI with the pruner sidecar running (Prometheus,
  `node_filesystem_avail_bytes{instance="192.168.101.11:9100"}`). The
  32 `Evicted` runner Pods left from 2026-09-19 were deleted by hand;
  zero failed Pods remain. Decided with Julian: no automatic cleanup
  (descheduler, lower `terminated-pod-gc-threshold`) and no
  `ephemeral-storage` requests for now -- if evictions recur, fix the
  cause first, then delete the leftovers again. Standing caveat:
  changing a runner's hostPath cache path orphans the old directories
  on node-2; nothing cleans them up, so any such change needs a
  matching manual cleanup there.
- **`k3s_node` gained live VM-resize support** (`ansible/roles/k3s_node/
  tasks/provision_vm.yml`) after the vCPU/memory/disk bumps tracked in
  `PARKED.md`'s "k3s node VM memory bump" entry exposed that the role
  could only ever apply those values at first creation -- re-applying
  against an already-running VM was a silent no-op each time (missed
  once for vCPU-only comparisons, then again for memory, since neither
  the domain's `<vcpu>`/`<memory>` XML nor the qcow2's own virtual size
  have a live hotplug path here). The role now reads the existing
  domain's current vCPU/memory/disk size, shuts the VM down only if one
  of them actually needs to change, and refuses outright to shrink a
  disk (that would truncate the guest's own filesystem). Two real bugs
  along the way: `regex_search`'s backreference argument gets mangled
  by Jinja's own string-literal parsing before reaching the filter
  (`AnsibleFilterError: Unknown argument` regardless of escaping) --
  worked around by matching the whole `<vcpu>N</vcpu>` element and
  stripping the tags with two plain `regex_replace` calls instead of a
  capture group; and the first real resize's own shutdown-wait budget
  (5 minutes) wasn't enough for a real graceful ACPI shutdown under
  normal load, causing a several-minute outage before a second run
  picked up where the first left off -- widened to 15 minutes.
  Separately, `community.libvirt.virt_net`'s own `autostart: true` is a
  known upstream bug (ansible-collections/community.libvirt#107,
  ansible/ansible#27064) that can silently leave a network's real
  autostart flag at `no` despite the task reporting success, which is
  what left both k3s VMs shut off after a real host reboot -- worked
  around with a fallback to plain `virsh net-autostart`, unaffected by
  either issue.
- **`k3s_node`'s guest configuration disables systemd-resolved's stub
  listener** (`ansible/roles/k3s_node/tasks/configure_guest.yml`) after
  a real containerd DNS failure: Debian's default setup points
  `/etc/resolv.conf` at systemd-resolved's own loopback stub
  (`127.0.0.53`); glibc's NSS resolver talks to systemd-resolved's
  smarter D-Bus/Varlink interface regardless and never sees a problem,
  but Go binaries (containerd included) use Go's own pure-Go resolver,
  which fires a raw UDP query straight at whatever's in
  `/etc/resolv.conf`, bypassing that smarter path. With a real, working
  network underneath (a full TLS handshake reaching `ghcr.io` directly
  succeeded), containerd's own image pulls still failed repeatedly with
  `lookup ghcr.io: Try again` -- the stub listener itself was the
  problem, not any cached process state, so a full k3s/containerd
  restart never cleared it. Fixed with the standard, documented fix for
  this class of bug: disable the stub listener and point
  `/etc/resolv.conf` at systemd-resolved's own always-maintained
  real-nameserver file instead. The task restarting systemd-resolved
  checks the stub listener's actual live state directly (`wait_for` on
  `127.0.0.53:53`), not just whether this run's own tasks reported a
  change -- a run whose config already matched from an earlier apply
  still left the stub listening, because that earlier apply's own
  restart had never actually happened either.
- **`k3s_ingress_forward`'s iptables rules went through two real hairpin
  bugs**, both found live and both fixed with a one-time reconciliation
  task that removes the old rule shape (iptables' own `-C`/module
  matching is by exact parameter set, so simply applying a corrected
  rule leaves the old, broken one in place alongside it, still winning
  since DNAT terminates further `PREROUTING` processing). First: every
  DNAT rule this role applied had no `in_interface` restriction, so it
  matched the k3s VM's own *outbound* traffic re-entering `PREROUTING`
  too, hairpinning it back into the cluster's own Traefik instead of
  reaching the real internet. Second, a few days later: even with that
  interface restriction, the rule still had no destination filter at
  all, matching this *homeserver's own* locally-originated traffic on
  the same ports -- Nextcloud AIO's own mastercontainer, validating
  connectivity to the real `ghcr.io` on port 443, got silently
  hairpinned into this cluster's Traefik instead, completing a real TLS
  handshake with a real but wrong certificate
  (`"SSL: no alternative certificate subject name matches target
  hostname 'ghcr.io'"`) and crash-looping for hours before the cause
  was found. Fixed by also restricting the DNAT rule to
  `destination: "{{ ansible_host }}"`. Separately, the base `FORWARD`
  chain's own `ACCEPT` rule for this traffic has to be inserted ahead
  of both Docker's `DOCKER-USER`/`DOCKER-FORWARD` chains and libvirt's
  own `LIBVIRT_FWX`/`FWI`/`FWO` chains -- libvirt's auto-managed
  `LIBVIRT_FWI` chain only accepts `RELATED,ESTABLISHED` traffic into a
  NAT-mode network by default, rejecting a fresh externally-initiated
  connection otherwise. A third, separate incident: `state` on both
  iptables tasks flips between `present`/`absent` directly off
  `k3s_ingress_forward_enabled`, a real toggle rather than a skip-guard
  -- before that variable was persisted in inventory, one ordinary
  `ansible-playbook` invocation that forgot the `-e
  k3s_ingress_forward_enabled=true` flag defaulted to `false` and
  actively tore out the live 80/443/53 DNAT rules, a real production
  outage for every public `*.jkandler.de` service. A real toggle needs
  its own real, persisted state; relying on a manually-remembered `-e`
  flag every apply was the actual bug.

## Mail server (Stalwart/Bulwark)

Self-hosted mail for `info@`/`julian@jkandler.de`, built 2026-09-19/22
on the k3s cluster (`infra/k3s-apps`' `modules/stalwart`, `modules/
bulwark`, `modules/authelia`, `modules/stalwart_config`) rather than
AWS, after a real port-25 test confirmed the ISP doesn't block it.
Chosen over docker-mailserver + Nextcloud (weighed and rejected: two
servers, two credentials, no "one unit" feel for mail/calendar/
contacts). See `docs/home-infra-docs/docs/runbooks/stalwart-mail-server.md`
for the operational runbook -- several Stalwart settings (CORS, IP
allow-list, outbound relay route, DKIM) only take effect after a Pod
restart, confirmed the hard way three times during this build. Two
Deployment-level gotchas found building `modules/stalwart`: until the
bootstrap wizard is completed, Stalwart listens on port 8080 only (no
SMTP listener at all), so the original readiness/liveness probes on
port 25 failed forever and restarted the Pod every ~3 minutes -- fixed
with a startup-only probe on 8080, steady-state probes left for a
follow-up once the real post-setup listener set is known. The memory
limit sits close to the setup wizard's own RocksDB defaults (128MB
write buffers + 128MB block cache, ~256MB before Stalwart's own
process even runs), which is why it's sized well above that ceiling
rather than against measured usage.

- **Inbound**: port 25 only, via `home-infra`'s `k3s_ingress_forward`
  DNAT (`home-infra#51`). **Outbound**: relayed through `aws/ses-relay`'s
  SES identity. **DNS**: `aws/dyndns`/`aws/ses-relay` own the MX/SPF/
  DMARC/`mail`/`stalwart` subdomain records. A real Outlook round-trip
  (send in, reply out) delivered cleanly with DKIM/SPF/DMARC all
  passing. SES production access was granted 2026-09-21 (50,000
  messages/day, eu-central-1).
- **Webmail + SSO**: Bulwark (`mail.jkandler.de`) with full Authelia SSO
  into the real mailbox -- no separate Stalwart password. Solved the
  SSO-vs-admin-console conflict by finding that a Stalwart API key
  (Bearer token) authenticates against the management API regardless of
  the "Authentication Directory" setting, since it's a Stalwart-internal
  credential, not routed through the configured directory -- so the
  directory can stay permanently on Authelia SSO, with admin changes
  going through the API (key in `pass` as `mail/admin/api`) instead of
  toggling the directory back and forth.
- **Mail clients**: IMAP 993 / submission 465 forwarded end to end
  (`home-infra#53`, `k3s-apps#79`), Stalwart serving a real Let's
  Encrypt cert via its own ACME (HTTP-01, managed as code --
  `k3s-apps#82`). App passwords and Authelia SSO coexist (an app
  password authenticated over IMAP with SSO on). Outlook for iOS can't
  do custom SSO, only IMAP + app password, so it uses that path; Apple
  Mail connects directly. No backup MX -- a homeserver outage delays
  inbound mail (senders retry for days) rather than losing it, judged
  not worth building for.
- **Stalwart settings as code**: started 2026-09-21 via the community
  `tahacodes/stalwart` Terraform provider (v0.2.4, pre-1.0, pinned
  exactly), authenticating with a dedicated admin API key
  (`k3s-apps/stalwart` in Secrets Manager). Chosen over a separate repo
  against the usual advice: a Stalwart outage makes every `k3s-apps`
  plan fail (including the one that would fix it), mitigated with
  `terraform plan -target=module.stalwart_config`. Applied so far: CORS
  and the pod-network allowed IP, the ACME provider, and the adopted
  mail domain (DKIM set to `Manual`, since Stalwart's own DKIM and
  SES's Easy DKIM would otherwise double-sign). Still manual: the
  Authentication Directory/SSO OIDC directory setting itself (riskiest
  to get wrong -- a mistake locks out everything but the API key), the
  SES relay route, IP filters.
- **Known gaps, not yet closed**: no off-server backup of mail data (the
  k3s VM's `local-path` PVC only); SPF is soft-fail (`~all`) deliberately
  while outbound is new, to be tightened to `-all` once it's run clean
  for a week or two; three Authelia secrets (`authelia_session_secret`,
  `authelia_storage_encryption_key`, `authelia_oidc_issuer_private_key`)
  were accidentally exposed to an assistant session during diagnostics
  and still need rotating.

## AWS automation identity

- the local-only `repo-infra` Terraform root (renamed from `github-infra`)
  owns the account-wide GitHub OIDC provider and the DynDNS plan/apply roles
  and inline policies, alongside GitHub repository settings management
- its state is on the shared encrypted S3 backend under
  `repo-infra/terraform.tfstate`
- DynDNS PR #8 merged the original no-destroy source-state handoff (to the
  now-deleted `aws-account-bootstrap`) as `d8d6541` on 2026-08-19; on
  2026-08-21 the same 5 resources were migrated again into `repo-infra` using
  the identical no-destroy pattern, verified by a clean `terraform plan`
  (only the 5 imports plus 2 cosmetic IAM description changes) and a
  post-migration `gh api` check confirming `dyndns`'s live Actions variables
  still match the real role ARNs
- the control-plane root must run only locally with a human-selected,
  short-lived AWS identity plus a GitHub token, never in GitHub Actions or
  through a role it manages
- that local identity was AWS root throughout most of this workspace's
  history (MFA-enabled, no long-lived keys, so not an active leak, but
  broader than routine work needs). **Resolved 2026-08-24** (ADR 0018,
  `terraform-state`'s `operator.tf`): day-to-day `repo-infra` work now
  uses the existing IAM user `julian` (MFA enrolled, console login via
  `aws login`) instead of root. `julian` has a write policy scoped to
  exactly what running `repo-infra` needs (the shared state bucket's
  `repo-infra/*` prefix, the GitHub OIDC provider, per-repo deploy/plan
  roles) plus AWS's managed `ViewOnlyAccess` for ad-hoc read-only
  verification — deliberately not `ReadOnlyAccess`, since `ViewOnlyAccess`
  excludes any action that returns actual data, so it can't read this
  root's own state or any secrets. `julian` has zero IAM permissions over
  itself or `terraform-state`'s own state, so widening its access always
  needs a deliberate apply of `terraform-state`, run as root — that stays
  the only routine use of root going forward. Two real gaps were found and
  fixed live during rollout: `aws login`'s OAuth flow for a non-root IAM
  user needs two specific `signin:*` actions root never needed (missed
  initially, since root bypasses IAM checks entirely and the gap was only
  visible once `AdministratorAccess` was actually detached); and the
  initial write-only policy genuinely couldn't do the cross-service
  read-only checks (budgets, health checks, alarms, IAM audits) root had
  been used for throughout this workspace's history, fixed by adding
  `ViewOnlyAccess`. (Originally built as a separate
  `bootstrap/operator-identity` root, then merged into `terraform-state`
  the same day — both are foundational, root-only, essentially-
  bootstrap-once resources, so a third such root was pure overhead.)

## Known gaps

- no Nextcloud indexing; retrieval remains explicit and bounded (reads are
  read-only; file writes, ADR 0011, and Shopping List item writes, ADR
  0014, are both implemented but not yet deployed, and both execute
  immediately per ADR 0013 rather than behind a human confirmation step)
- no controlled Git tools exposed to the deployed agent
- no SMART or bounded log-inspection tool yet
- **fully resolved**: a 2026-08-22 homeserver audit (prompted by finding the
  un-tracked sankey script) found `iperf3` running as an unauthenticated
  server on all interfaces, five stray exited Docker containers, a dead
  `aws-bootstrap` Terraform leftover, and three unused ownCloud Docker
  volumes. All four are now applied/removed/deleted and confirmed gone live
  by the user.
- **resolved 2026-08-23**: a disaster-recovery audit found that 5 of this
  workspace's 7 git repositories (`home-infra`, `repo-infra`,
  `terraform-state`, `home-infra-docs`, `home-infra-ai-context`) existed
  only on one laptop, with no backup anywhere -- a total-loss risk for the
  entire Ansible source of truth if that one machine failed. All 5 now have
  a private GitHub backup mirror (`repo-infra`'s `modules/repo` gained
  `visibility`/`branch_protection_enabled` options for this — see its own
  section above), verified byte-for-byte identical to local `HEAD`. Full
  git history was scanned across all 5 repos first and confirmed free of
  real credentials or IP addresses before anything was pushed.
- its ignored non-empty local state backup is mode `0664`; contents were not
  inspected and require a recovery-aware security review
- **resolved 2026-08-23**: the SOPS/GPG decryption key
  (`6D8B16CB662983A54B4AF1466F0B5C2AB1509600`, created 2026-08-13) had
  existed only on that same one laptop with no backup, found by the same
  audit as the git-repo backups above. Fixed with an offline custody model
  rather than a second live key: the private+public key were exported to a
  freshly wiped and FAT32-formatted USB stick (`gpg --armor
  --export-secret-keys`/`--export`, plus a README with restore
  instructions but no passphrase), verified with a real restore drill
  (import into a throwaway `GNUPGHOME`, decrypt the actual
  `secrets.sops.yml`, confirmed working) before being handed to a family
  member. The user plans to periodically re-verify it and add a second USB
  copy with another family member. The private key export is exactly as
  protected as the key was on the laptop — it's still encrypted with its
  own passphrase, which only exists in the user's memory (also verified
  correct via `sops -d`/`gpg --clearsign` drills, run after killing
  `gpg-agent` to force a fresh prompt rather than a cached one).
- Nextcloud's actual data (files, photos, Deluge downloads, ~556GB on one
  HDD with no redundancy) still has zero backup — no BorgBackup container
  configured despite AIO supporting it natively, nothing off-disk. The
  user explicitly deferred this (**"can be ignored for now"**) after the
  2026-08-23 audit found it; not forgotten, just not being worked on.
  `docs/runbooks/rebuild-homeserver.md` already flagged this gap before
  the audit. **Concrete plan for whenever this is picked back up** (the
  user has a spare HDD on hand): enable AIO's built-in BorgBackup to that
  second local disk first (application-consistent, deduplicated backups of
  files + database together — cheap, immediate protection against the
  current single-disk-failure risk), then layer genuine off-site coverage
  by syncing the resulting Borg repository to S3 on a schedule (same timer
  pattern already used for the sankey exporter; Borg's deduplication keeps
  the synced footprint well under the raw 556GB, and S3 lifecycle rules to
  Glacier keep ongoing cost low). Explicitly parked, not started.
