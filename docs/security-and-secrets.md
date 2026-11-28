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

### Nextcloud on k3s

Since ADR 0033 Nextcloud runs in `apps` under PodSecurity `restricted`,
rootless as `www-data` (33); AIO's master container, which needed the
Docker socket, is stopped (parked until AIO is removed). The public route
has no forward-auth, like AIO's had: sync clients and apps sign in with
app passwords, the web login goes only through Authelia (force-SSO). A
NetworkPolicy admits Traefik, sankey-export and the backup Job only;
PostgreSQL and Redis live in the same Pod.

### Self-hosted GitHub Actions runners

**Updated 2026-11-27.** Only repositories whose CI needs the home network
get a self-hosted runner: today `stalwart-config` (Stalwart's in-cluster
API). Every other repository's CI, including every Docker build and
`container:` job, runs on GitHub-hosted runners since `gha-common`
v2.0.0 (2026-11-26).

The runner runs in k3s ([`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)'s
`platform/apps/arc/`, synced by the platform Argo CD, ARC since 2026-11-01, [ADR 0027](adr/0027-ci-runners-autoscale-with-arc.md))
on a dedicated, tainted node (`k3s-node-2`, `ci=github-runner:NoSchedule`).
ARC creates one Pod per job and deletes it afterwards. The runner Pod is
`ghcr.io/actions/actions-runner` as uid 1001, without Docker, root or
hostPath, and `ci` enforces PSA `restricted`; no runner mounts a
ServiceAccount token. The controller and its listeners (`arc-system`,
`enforce: restricted`) run on `k3s-node-1`, so nothing on the node that
runs CI code holds the GitHub App key or the controller's right to create
Pods in `ci`. A NetworkPolicy isolates every Pod in `ci`: no inbound,
outbound only DNS and the public internet, plus Stalwart's API for its
runner.

Until 2026-11-26 every runner Pod had a privileged `docker:dind`
sidecar, a hostPath Terraform plugin cache and a GHCR pull-through cache,
and ran for nine repositories. A privileged container can escape to its
node, which made a CI job a path to that node's root, and the agents
then held the k3s server token. Both are gone: the runners lost Docker
and root, and the agents join with their own agent token.

Runners are ephemeral per Pod: every job starts in a fresh Pod with only
an `emptyDir` work volume, so no job sees another job's checkout or tool
cache.

Runner registration used a shared, manually-rotated fine-grained PAT
until 2026-09-16, when it switched to GitHub App installation tokens
(`kandlerli-home-infra-runner`, `Administration: write`, installed on
every runner-enabled repository). Since 2026-11-01 the ARC controller
and listeners use the App (Kubernetes Secret `arc-github-app` in `ci`) to
get a one-time JIT registration for each runner Pod. The runner Pods
never see the App key.

## Public exposure controls

Every public route, its auth and its limits: [current-state](current-state.md)'s
public-entry table; the rules are in the `check-security-invariants` skill.

## Data sent to cloud services

No AI provider receives homeserver or Nextcloud data: the agent, Open WebUI
and their Nextcloud tools were retired on 2026-11-16
([decisions](decisions.md#open-webui-and-home_agent-retired)).
