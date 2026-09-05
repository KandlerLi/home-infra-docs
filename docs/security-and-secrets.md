# Security and Secrets

## Secret management

The infrastructure repository uses SOPS with a GPG recipient. Ansible loads
encrypted variables through the `community.sops.sops` vars plugin.

Tracked encrypted file:

```text
ansible/inventory/group_vars/all/secrets.sops.yml
```

Current encrypted variable names:

- `github_runner_github_token`
- `home_agent_openai_api_key`
- `shared_ingress_agent_auth_password`
- `shared_ingress_agent_auth_password_hash`
- `shared_ingress_deluge_auth_password_hash`

Secret values must never be copied into this repository, defaults files,
Docker image layers, Compose files, command output, or architecture diagrams.

### Terraform state

`/home/julian/projects/bootstrap/terraform-state` is a dedicated Git repository that owns
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

`/home/julian/projects/github/repo-infra` (renamed from `github-infra`; absorbed the
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
- SOPS decryption occurs on the controller.

### Nextcloud AIO to Docker

The AIO master container receives the Docker socket because it manages sibling
containers. This makes it a highly trusted component. A read-only bind mount
of a Unix socket does not make the Docker API itself harmless, so access must
not be extended to unrelated containers.

### Physical host to runner VM

The runner is isolated in a Debian VM on a private NAT network. It receives no
host Docker socket, host SSH private key, or HDD mount. Separate runner Unix
accounts reduce accidental repository crossover inside the VM, but do not
provide kernel-level isolation from each other.

### Deployed AI boundary

The deployed design separates the model-facing container from host access:

```text
Cloud model
   |
home-agent container (unprivileged)
   |
restricted Unix-socket API
   |
home-tools service (fixed read-only operations)
```

The agent has no arbitrary shell, unrestricted SSH, root account,
Docker socket, autonomous Ansible deployment, file deletion, or package
installation capability.

## Public exposure controls

The agent API is published at `ai.jkandler.de` with:

- HTTPS termination
- authentication independent of the model prompt
- rate and request-size limits
- explicit proxy trust configuration
- security headers appropriate to the UI/API
- audit-friendly logs that exclude prompts, secret values, and tool payloads
- verified preservation of the current Nextcloud route

The shared proxy, Nextcloud reverse-proxy migration, guarded rollback path,
agent authentication, and limits are deployed. The Basic Auth username is
`julian`; the password and verifier remain encrypted with SOPS. The underlying
agent port remains bound to host loopback.

### Private Open WebUI boundary

The privately deployed Open WebUI keeps the existing home-agent as the security
and provider boundary. Open WebUI receives a placeholder API key, not
`home_agent_openai_api_key`, and can reach only the agent's compatibility API
over a non-masqueraded Docker bridge. It receives no Docker socket, Nextcloud
mount, host tool socket, shell, or SSH credential. Live validation confirmed
that outbound HTTPS is blocked.

Uploads, plugins, runtime package installation, code execution, the code
interpreter, web search, image generation, API keys, subagents, and workspace
tools are disabled. The image is pinned by tag and digest, runs under a
non-login host UID with all capabilities dropped and `no-new-privileges`, and
has memory, CPU, process, and log limits. Its upstream startup behaviour
requires a writable container root filesystem; this residual risk is recorded
and must be reconsidered on each image upgrade.

Open WebUI introduces sensitive local account and chat history under
`/var/lib/open-webui`. Back it up before upgrades, protect backups as private
data, and do not copy chats into logs or documentation. The public UI relies on
Open WebUI authentication; stacking Traefik Basic Auth would collide with the
Bearer `Authorization` header used by signed-in Open WebUI requests. The exact
legacy `/healthz` and `/v1/chat` routes retain Traefik Basic Auth.

The first administrator was created privately on 2026-08-19. A post-onboarding
check confirmed authentication and login enabled, onboarding closed, and
anonymous signup disabled. The guarded public route was enabled on 2026-08-19;
authenticated public login/chat was user-confirmed, and inventory flags were
persisted.

## Data sent to cloud services

When requested, selected health data such as load, disk usage, container
names, and container health state is sent to the configured OpenAI model.
Nextcloud documents are explicitly outside the deployed milestone and must
remain opt-in in any future retrieval design.

The private web UI does not change that rule. Any future Nextcloud integration
must use a separate least-privilege service, dedicated credentials, scoped
paths and operations, explicit confirmation for writes, and audit records that
do not disclose document contents.

### Prepared read-only Nextcloud boundary

Infrastructure revision `313283c` prepares but does not deploy that separate
boundary. `nextcloud-tools.service` is a dedicated non-login identity with no
Docker group, no capabilities, a strict protected filesystem, and network
access restricted to Unix sockets plus `127.0.0.1`. It authenticates to the AIO
Apache loopback endpoint with a service-owned `0400` app-password file.

The future `home-agent` integration mounts only the protected Unix socket and
adds its numeric client group. Neither `home-agent` nor Open WebUI receives the
app password, Nextcloud data mount, or general WebDAV network client. The
service supports only bounded `PROPFIND` metadata and small approved text-file
`GET` requests below one configured root. It contains no WebDAV write methods.

The dedicated Nextcloud account and app password do not yet exist. The guarded
bootstrap creates the non-admin account with an unrevealed generated login
password and creates a limited app password without supplying that login
password. The token exists in non-cached Ansible memory only during the apply;
all relevant tasks use `no_log`, and the final value is written directly as the
service-owned `0400` host file. It is not stored in SOPS or Docker.

The human owner must share only `AI Workspace` read-only with the new account.
The guarded rotation play validates a replacement over loopback WebDAV before
installing it and revokes only previous tokens named `home-agent-readonly`.
Revoke the token independently if compromise is suspected.

Requested filenames, metadata, and text contents cross the cloud boundary when
the agent calls these tools. There is no background indexing. Populate the
allowed folder only with data that may be disclosed to the configured provider
on explicit request.
