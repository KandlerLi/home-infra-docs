# Current State

- Snapshot date: **2026-08-19**
- Production host: `homeserver` (`192.168.178.100`)

## Executive summary

The physical homeserver is running Debian 13.6. Docker and libvirt are active.
Nextcloud AIO is the primary production application and all ten observed AIO
containers report healthy. The large HDD is mounted and has substantial free
capacity. No failed systemd units were observed.

The physical RAM was upgraded from 8 GiB to 16 GiB on 2026-08-19. After the
maintenance boot, Debian reported all 16 GiB online, approximately 15.5 GiB
usable, and no offline blocks or memory-related hardware errors in the current
boot's accessible kernel journal. The AIO application containers remained
stopped after boot and were started manually through the AIO interface; their
final state and public endpoint were healthy.

The GitHub Actions runner is configured by committed Ansible code. The
dedicated bridge, guest interface, static neighbour entry, and VNC listener
were observed. A non-privileged `virsh` query timed out, so the libvirt domain
state was not independently enumerated during this snapshot.

The home-agent stack is deployed and verified. The `home-tools` systemd service
is active, the `home-agent` Docker container is running, and the container API
remains bound to loopback. Traefik publishes it at `ai.jkandler.de` behind
Basic Auth, rate and request-size limits, and security headers.

Open WebUI is privately deployed and healthy on `127.0.0.1:8091`. Its
non-masqueraded Docker bridge blocks external egress while allowing it to reach
only `home-agent` and publish on host loopback. The initial administrator has
not been created and the public hostname still routes directly to the JSON
home-agent API, so a public browser request to `/` does not yet show a chat
window.

Traefik `v3.7.1` is the shared production ingress. It owns TCP ports 80 and
443, obtains certificates through ACME, and routes `nextcloud.jkandler.de` to
Nextcloud AIO Apache on `127.0.0.1:11000`. The migration completed and was
verified on 2026-08-19. The authenticated agent route is enabled.

## Live evidence

| Area | Observed state |
| --- | --- |
| Operating system | Debian 13.6 (`trixie`) |
| Kernel | `6.12.101+deb13-amd64` |
| CPU | 4 logical CPUs; Intel Core i5-2400 documented hardware |
| Memory | 16 GiB online; 15.5 GiB usable; no offline blocks |
| Root filesystem | 51 GiB total, 26 GiB used, 23 GiB available, 54% used |
| Large HDD | 916 GiB total, 138 GiB used, 732 GiB available, 16% used |
| Failed systemd units | None observed |
| Docker service | Active |
| libvirt service | Active |
| Nextcloud AIO | Ten running containers, all reporting healthy |
| `home-tools` service | Active; fixed read-only collectors over a protected Unix socket |
| `home-agent` container | Running; healthy API on `127.0.0.1:8090` |
| Open WebUI container | Running and healthy on `127.0.0.1:8091`; private onboarding pending |
| Frontend network | `home-agent-frontend`; IP masquerading disabled; only Open WebUI and `home-agent` attached |
| Agent end-to-end check | Successful OpenAI tool call and health report |
| Public agent boundary | Valid Let's Encrypt certificate; unauthenticated 401; authenticated health and chat HTTP 200 |
| Shared ingress | Traefik `v3.7.1`, healthy on ports 80/443; no Docker socket |
| Nextcloud upstream | AIO Apache on `127.0.0.1:11000` only |
| Public Nextcloud check | HTTP/2 200; maintenance disabled; valid Let's Encrypt certificate |
| Runner network | `virbr10` active at `192.168.100.1/24` |
| Runner guest evidence | `vnet9` present; `192.168.100.10` reachable in neighbour table |
| AWS identity control plane | OIDC provider and DynDNS plan/apply roles owned by local-only `aws-account-bootstrap`; migration completed |
| DynDNS automation after handoff | PR #8 checks and protected apply succeeded; final no-change plans and role tests user-confirmed |

Disk values above are human-readable `df` values and may differ slightly from
byte-level filesystem statistics.

## Repository state

Infrastructure repository root:

```text
/home/julian/projects/home-infra
```

Current committed revision at snapshot time:

```text
9c97e88
Fix Open WebUI private network startup
```

This revision contains the deployed private Open WebUI implementation. The
public Open WebUI route is still intentionally disabled.

DNS infrastructure repository root and current revision:

```text
/home/julian/projects/dyndns
b44f17a
Remove completed state handoff scaffolding (#9)
```

Merged Terraform change `0fde1dd449039abddd2de6c34c3075569f773703`
introduced the `ai` CNAME declaration.

AI handover repository root and orientation revision before this update:

```text
/home/julian/projects/home-infra-ai-context
e6ab855
Record Open WebUI preparation handover
```

This repository is a secret-free orientation layer for new models, chats, and
tasks. It points back to the implementation and C4 sources of truth.

Terraform state-bucket bootstrap project:

```text
/home/julian/projects/terraform-state
```

This is a Git repository on `main`, initialized at revision `fbac0af`. It manages
the versioned, encrypted, public-access-blocked `jkandler-terraform-state` S3
bucket and uses that bucket for its own `terraform-state/terraform.tfstate`
backend key. Its tracked initial commit contains only Terraform source, the
provider lock file, ignore rules, repository instructions, and documentation.
Ignored local state artifacts remain: the root state file is empty, while a
non-empty backup and local backend metadata were observed with mode `0664`.
Their contents were not inspected. Recovery verification, tighter local
permissions, and remote repository protection remain follow-up work.

AWS account bootstrap project:

```text
/home/julian/projects/aws-account-bootstrap
1a42299
Remove completed import scaffolding
```

This local Git repository contains the deployed local-only Terraform root
for the account-wide GitHub Actions OIDC provider and the DynDNS plan/apply
roles and policies. Its remote backend key is
`aws-account-bootstrap/terraform.tfstate`.

The state handoff completed on 2026-08-19. DynDNS PR #8 merged as `d8d6541`;
its validation, speculative plan, exact-commit approval, and protected apply
succeeded. The five source bindings were removed with `destroy = false`, then
the existing resources were imported through the local bootstrap root. The
user confirmed final no-change plans and successful plan/apply role tests.

The committed repository contains these roles:

- `base`
- `storage`
- `docker`
- `nextcloud_aio`
- `libvirt_host`
- `github_runner`
- `home_agent`
- `open_webui` (privately deployed; public route disabled)
- `shared_ingress`

The `home_agent` role, dedicated playbook, tests, and encrypted API credential
are committed. The dedicated playbook was applied with human-entered sudo
credentials on 2026-08-18.

The guarded Nextcloud migration completed on 2026-08-19. The deployed state is
persisted with Nextcloud reverse-proxy mode, shared ingress, and the
authenticated agent router enabled. Traefik is running and healthy.

## Managed services

### Nextcloud AIO

Observed running containers:

- `nextcloud-aio-mastercontainer`
- `nextcloud-aio-apache`
- `nextcloud-aio-nextcloud`
- `nextcloud-aio-database`
- `nextcloud-aio-redis`
- `nextcloud-aio-imaginary`
- `nextcloud-aio-whiteboard`
- `nextcloud-aio-notify-push`
- `nextcloud-aio-talk`
- `nextcloud-aio-collabora`

The AIO master container is managed directly by Ansible. AIO itself creates
and manages the sibling containers.

### GitHub Actions runner

Committed automation defines a Debian 13 VM with:

- 2 virtual CPUs
- 2048 MiB RAM
- a 40 GiB qcow2 disk
- static address `192.168.100.10`
- private NAT network `192.168.100.0/24`
- one runner installation and Unix account per configured repository

The currently configured repository is `KandlerLi/dyndns`.

## Known gaps and drift candidates

| Observation | Classification | Follow-up |
| --- | --- | --- |
| No monitoring role or monitoring containers | Current gap | Design lightweight monitoring separately |
| `iperf3`-typical port `5201` is listening | Unknown | Identify owner and decide whether to manage or remove it |
| Loopback port `43117` is listening | Unknown | Identify owning process with privileged diagnostics if needed |
| Legacy `owncloud-docker-server_*` Docker volumes exist | Unknown/legacy | Do not delete until contents and recovery need are confirmed |
| One anonymous Docker volume exists | Unknown | Identify owner without exposing stored data |
| Local `.venv` contains the obsolete repository path | Controller issue | Recreate the virtual environment at the current path |
| `terraform-state` has no remote repository or hosting protections | Repository-governance gap | Publish and secure it deliberately after verifying the recovery/exclusion model |
| Non-empty ignored Terraform state backup has mode `0664` | Local security gap | Do not read/delete; verify recovery need and tighten permissions with approval |

## Explicitly not deployed

- public Open WebUI route at `ai.jkandler.de`
- Nextcloud document retrieval or indexing

The private deployment added a non-masqueraded frontend bridge, an
OpenAI-compatible endpoint on `home-agent`, and a loopback-only Open WebUI
service on port 8091. The guarded publish/rollback playbooks exist, but the
public route was not changed.

## Agent verification evidence

Observed initially on 2026-08-18 and revalidated publicly on 2026-08-19:

- `home-tools.service` active under systemd
- `home-agent` running with a read-only root filesystem
- Docker privileged mode disabled and every Linux capability dropped
- only the protected tool socket directory and API-key file mounted read-only
- host publication restricted to `127.0.0.1:8090`
- live health request reported `/` at 50.9% and `/mnt/black-hdd` at 15.0%
- no failed systemd units reported
- `ai.jkandler.de` resolves by CNAME to `jkandler.de` with IPv4 and no AAAA
- unauthenticated public requests return HTTP 401
- authenticated `/healthz` and `/v1/chat` requests return HTTP 200
- Open WebUI is healthy on `127.0.0.1:8091`, runs as UID `995` with GID `0`,
  has all capabilities dropped, and has no Docker socket
- Open WebUI can list the synthetic `home-agent` model over the dedicated
  bridge, while an outbound HTTPS test from the container is blocked
- a live compatibility completion returned exactly `compatibility-ok`
