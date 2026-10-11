# Security and Secrets

## Secret management

**Rewritten 2026-09-16: this section previously described SOPS, which
was fully retired 2026-09-09.** Every secret in this workspace now
lives in AWS Secrets Manager, managed by a dedicated repository
([`secrets-manager`](https://github.com/KandlerLi/secrets-manager)) whose own Terraform creates only the secret
*containers* — never a value. Each container is a self-contained
module directory (`secrets/<repo>/<service>/`, its own `main.tf` plus
a `README.md` documenting exactly which keys it holds, who consumes
them, and how to rotate each). Real values are always set out-of-band
via `aws secretsmanager put-secret-value`, never through Terraform
(there is deliberately no `aws_secretsmanager_secret_version` resource
anywhere in this workspace — putting a real secret value in Terraform
state is treated as equivalent to committing it in plaintext).

Every repository that needs a secret reads it directly from AWS
Secrets Manager at plan/apply time — Ansible's `site.yml` does this in
its own `pre_tasks` via the `amazon.aws.secretsmanager_secret` lookup
plugin; each Terraform repository's own `secrets.tf` does it via a
`data "aws_secretsmanager_secret_version"` + `jsondecode()`, one data
source per secret group. There is no tracked encrypted file anywhere
in this repository any more, and no local decryption step — reading a
secret requires only the caller's own AWS credentials (an `aws login`
session for a human, an IAM role for CI), the same authentication
every other AWS API call in this workspace already needs.

Secret values must never be copied into any repository, defaults
files, Docker image layers, Compose/Kubernetes manifests, command
output, or architecture diagrams — this rule didn't change with the
mechanism.

### Terraform state

[`terraform-state`](https://github.com/KandlerLi/terraform-state) is a dedicated Git repository that owns
the central S3 state-bucket configuration. The bucket is configured with
versioning, AES-256 server-side encryption, and full public-access blocking.
Its ignore rules exclude state, backup, plan, variable-value, crash, override,
and `.terraform` working-directory artifacts.

Ignored local state artifacts remain from bootstrap/migration. Their contents
were not inspected. A non-empty backup and local backend metadata were observed
with mode `0664`, which permits local non-owner reads. Treat them as potentially
sensitive: do not open, copy, commit, upload, delete, or change them without a
recovery-aware, explicitly approved plan.

### Repository infrastructure: GitHub settings and AWS identity control plane

[`repo-infra`](https://github.com/KandlerLi/repo-infra) (renamed from `github-infra`; absorbed the
formerly separate `aws-account-bootstrap` root on 2026-08-21, see
[ADR 0010](adr/0010-consolidate-repo-and-identity-control-plane.md)) is a
local-only Terraform root (no GitHub remote of its own) that manages, per
repository declared in `config.yml`: GitHub-side settings (branch ruleset,
Actions allowed-actions allowlist and `sha_pinning_required`, repository
default workflow permissions, Actions variables, the `production`
environment — see
[ADR 0009](adr/0009-terraform-managed-github-repository-protection.md)) and,
optionally, AWS deploy credentials (the account-wide GitHub OIDC provider,
per-repository IAM plan/apply roles, and the Actions variables that wire
those roles' real ARNs into the repository).

The account-wide OIDC provider and `dyndns`'s IAM roles/policies were
absorbed from `aws-account-bootstrap` via the same no-destroy handoff pattern
established by ADR 0006: `removed { destroy = false }` blocks in the source
root, `import` blocks in the destination, a `terraform plan` showing only
imports (plus two purely cosmetic IAM role description changes — trust
policies were byte-identical) before applying. A post-migration `gh api`
cross-check confirmed `dyndns`'s live Actions variables still match the real
role ARNs; its CI required no changes. `aws-account-bootstrap` is now
deleted.

The control plane must run only with a human-selected, short-lived local
bootstrap identity, never through GitHub Actions or a role it manages. A
plan/apply role must never be able to change its own trust or permissions.
The same no-destroy rule remains mandatory for any future ownership transfer.
State lives in the shared encrypted S3 backend
(`repo-infra/terraform.tfstate`).

`whoami` has its own pre-existing manual GitHub protection (a disabled branch
ruleset, non-default merge settings, unrestricted Actions permissions) that
remains entirely outside Terraform and outside this documentation set,
pending a separate archive/delete decision for that repository.

## Trust boundaries

### Ansible controller to homeserver

- SSH authenticates the controller to `julian@192.168.178.100`.
- Ansible uses `/usr/bin/python3` remotely.
- Privileged roles use `become`.
- Secrets are read from AWS Secrets Manager at apply time (see "Secret
  management" above) — there is no local decryption step, unlike the
  retired SOPS mechanism this section used to describe.

### Nextcloud AIO to Docker

The AIO master container receives the Docker socket because it manages sibling
containers. This makes it a highly trusted component. A read-only bind mount
of a Unix socket does not make the Docker API itself harmless, so access must
not be extended to unrelated containers.

### Self-hosted GitHub Actions runners

**Rewritten 2026-09-16: this section previously described a single,
dedicated libvirt VM.** That VM was decommissioned 2026-08-31 in favor
of one k3s Deployment per repository ([`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)'s own
`modules/github_runner`), pinned to a dedicated, tainted node
(`k3s-node-2`, `ci=github-runner:NoSchedule`) so CI workloads never
compete with or endanger anything on `k3s-node-1`. Every repository in
[`repo-infra/config.yml`](https://github.com/KandlerLi/repo-infra/blob/main/config.yml)'s `runner: true` list gets its own
runner Pod, not just one shared VM serving a single repository.

Each Pod runs the community `myoung34/github-runner` image with a
privileged `docker:dind` sidecar — a deliberate trade-off, not an
oversight: it lets every repository's existing `docker build`/`docker
run` workflow steps keep working completely unmodified, at the real
cost that a privileged container can escape to its own node. This is
contained specifically to `k3s-node-2`, which runs nothing else, the
same isolation principle the old VM's own private NAT network
provided. Each Pod is ephemeral (one job, then a fresh Pod for the
next), sidestepping the whole class of cross-job workspace-reuse bug a
persistent runner needs real logic to handle.

Runner registration used a shared, manually-rotated fine-grained PAT
until 2026-09-16, when it switched to GitHub App installation tokens
(`kandlerli-home-infra-runner`, `Administration: write`, installed on
every runner-enabled repository) — the runner image mints its own
short-lived registration token internally at container start, rather
than a durable credential sitting in AWS Secrets Manager waiting to be
manually rotated.

### Deployed AI boundary

**Rewritten 2026-09-16: this section previously described a single
homeserver Docker container.** `home_agent` moved to a k3s-native
Deployment 2026-08-30 ([`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' `modules/home_agent`) and now
runs two containers as sidecars in one Pod, sharing a Unix socket
through an `emptyDir` volume — the same trust shape the pre-migration
homeserver deployment already had, just inside one Pod instead of one
host:

```text
Cloud model
   |
home-agent container (unprivileged, sidecar A)
   |
restricted Unix-socket API (shared emptyDir)
   |
nextcloud-tools container (unprivileged, sidecar B)
```

`home_tools_service` (which reports the physical homeserver's own
hardware health — disk, load, container status) couldn't move into
this Pod the way `nextcloud_tools` did, since it genuinely needs to
observe the real host, not a k3s VM. It's reached instead over a
narrow, opt-in TCP listener bound to the k3s VM's own isolated-network
address (`192.168.101.1:8095`), not a Unix socket, and not exposed
beyond that network.

The agent has no arbitrary shell, unrestricted SSH, root account,
Docker socket, autonomous Ansible/Terraform deployment, file deletion,
or package installation capability. `home_agent`'s own image is a
private GHCR package built by a dedicated CI workflow on the
self-hosted runner — deliberately never built on the homeserver
itself.

## Public exposure controls

**Rewritten 2026-09-16: authentication changed from Traefik Basic Auth
to Authelia forward-auth (2026-09-08), and routing moved from the
homeserver's own `shared_ingress` role to k3s-native Traefik
(2026-09-01).** The agent API is published at `ai.jkandler.de/healthz`
and `/v1/chat` with:

- HTTPS termination (k3s-native Traefik, ACME-issued certificates)
- Authelia forward-auth (MFA-gated OIDC session, not a static
  username/password pair — unauthenticated requests redirect to
  `auth.jkandler.de` rather than returning a bare 401)
- authentication independent of the model prompt
- rate and request-size limits
- explicit proxy trust configuration
- security headers appropriate to the UI/API
- audit-friendly logs that exclude prompts, secret values, and tool
  payloads

`ai.jkandler.de` is a single hostname split by *path* between two
separate k3s Deployments: `/healthz` and `/v1/chat` go to `home_agent`
(Authelia-gated as above); everything else goes to `open_webui`,
which deliberately does **not** carry Authelia forward-auth — it does
its own app-level Authelia OIDC login instead (see below). The
underlying `home_agent` Service is never reachable except through this
ingress chain — no direct `NodePort`/`hostPort`.

### Private Open WebUI boundary

**Rewritten 2026-09-16: this section previously described a
homeserver Docker container with a placeholder API key over a
non-masqueraded bridge.** That deployment stopped 2026-08-30; Open
WebUI now runs as its own k3s-native Deployment
([`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' `modules/open_webui`). Its connection to
`home_agent` is to that Deployment's own in-cluster Service now, not a
Docker bridge — Open WebUI's container still never gains real
outbound egress to anything but that Service, the same boundary the
old bridge enforced, just expressed differently. Speech-to-text (ADR
0012) is still routed through `home_agent`'s own bounded relay for the
same reason chat is.

Since 2026-09-08, Open WebUI's own login is Authelia OIDC SSO
(MFA-gated) — native email/password login and signup are both
disabled at the protocol level (`ENABLE_PASSWORD_AUTH`/
`ENABLE_SIGNUP=False`), closing what was otherwise a second,
MFA-skipping credential path sitting behind Authelia's own gate. There
is no Traefik Basic Auth in front of the UI route at all any more —
Open WebUI's own authentication is the only gate, which is also why
stacking Basic Auth in front of it was never viable: it would collide
with the `Authorization: Bearer` header signed-in Open WebUI requests
already carry.

Uploads, plugins, runtime package installation, code execution, the code
interpreter, web search, image generation, API keys, subagents, and workspace
tools are disabled. The image is pinned by tag and digest, runs under a
non-login UID with all capabilities dropped and `no-new-privileges`, and
has memory, CPU, process, and log limits. Its upstream startup behaviour
requires a writable container root filesystem; this residual risk is recorded
and must be reconsidered on each image upgrade.

Open WebUI's real account/chat-history data (`/var/lib/open-webui` on
the homeserver, NFS-exported into the k3s Pod — migrated live from the
original homeserver deployment, not started fresh) is sensitive local
data. Back it up before upgrades ([`home-infra`](https://github.com/KandlerLi/home-infra)'s `aux_backup`
role, added 2026-09-16, now covers this specifically), protect backups
as private data, and do not copy chats into logs or documentation.

The first administrator was created privately on 2026-08-19. A post-onboarding
check confirmed authentication and login enabled, onboarding closed, and
anonymous signup disabled. The guarded public route was enabled on 2026-08-19;
authenticated public login/chat was user-confirmed, and inventory flags were
persisted.

## Data sent to cloud services

When requested, selected health data such as load, disk usage, container
names, and container health state is sent to the configured OpenAI model.

**Corrected 2026-09-16: the read-only Nextcloud boundary this section
used to describe as "outside the deployed milestone" has actually
been live since 2026-08-21.** See "Nextcloud tool boundary" below for
its real, current scope. Write access (ADR 0011) and Shopping List
tool access (ADR 0014) remain implemented in code but not yet
deployed — both would execute immediately per ADR 0013 (no
human-confirmation step) once they are.

Any future Nextcloud integration beyond what's already deployed must
use a separate least-privilege service, dedicated credentials, scoped
paths and operations, and audit records that do not disclose document
contents. There is no background indexing today, and none should be
added without a separate threat review.

### Nextcloud tool boundary

**Rewritten 2026-09-16: this section previously described a
"prepared but not deployed" boundary.** It's been live since
2026-08-21, and the underlying deployment mechanism changed
2026-08-30 — `nextcloud_tools_service.py` now runs as a sidecar
alongside `home_agent` in the same k3s Pod (see "Deployed AI boundary"
above), sharing a Unix socket through an `emptyDir`, not a
homeserver-local Unix socket file. It authenticates to Nextcloud AIO's
Apache over the k3s VM's own isolated-network address
(`192.168.101.1:11000`), not homeserver loopback.

Neither `home_agent` nor Open WebUI receives the Nextcloud app
password, a Nextcloud data mount, or a general WebDAV network client.
The service supports only bounded `PROPFIND` metadata and small
approved text-file `GET` requests below one configured root. It
contains no WebDAV write methods in its currently-deployed form.

The dedicated Nextcloud account (`home-agent`, non-admin) and its app
password were created by a guarded, one-time bootstrap: the account's
own login password is generated and never displayed, and its limited
app password is created without ever supplying that login password.
The token existed in non-cached Ansible memory only during that
original apply (`no_log` throughout); the current value lives in AWS
Secrets Manager (`home-infra/home-agent`), put there via a deliberate
`put-secret-value` — the same role SOPS used to play before the
2026-09-09 SOPS-to-Secrets-Manager cutover, never in the agent
container's own environment or filesystem.

The human owner shares only `Shared/AI Workspace` (read-only, for the
currently-deployed read scope) with the dedicated account. A guarded
rotation play validates a replacement token over loopback WebDAV
before installing it, and revokes only previous tokens named
`home-agent-readonly`. Revoke the token independently if compromise is
suspected.

Requested filenames, metadata, and text contents cross the cloud
boundary when the agent calls these tools. There is no background
indexing. Populate the shared folder only with data that may be
disclosed to the configured provider on explicit request.

Write access (ADR 0011, create/update/delete/move) and Shopping List
tool access (ADR 0014) both reuse this same dedicated account and app
password, and both execute immediately per ADR 0013 — no
human-confirmation step. Both are in the deployed `home_agent` code;
each only works once Julian has shared the matching folder or list with
the account. Live confirmation is tracked in the private task list.
