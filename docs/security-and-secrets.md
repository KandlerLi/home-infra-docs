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

Secret values must never be copied into this repository, defaults files,
Docker image layers, Compose files, command output, or architecture diagrams.

### Terraform state

`/home/julian/projects/terraform-state` owns the central S3 state-bucket
configuration but is currently not version-controlled. The bucket is configured
with versioning, AES-256 server-side encryption, and full public-access blocking.

Ignored local state artifacts remain from bootstrap/migration. Their contents
were not inspected. A non-empty backup and local backend metadata were observed
with mode `0664`, which permits local non-owner reads. Treat them as potentially
sensitive: do not open, copy, commit, upload, delete, or change them without a
recovery-aware, explicitly approved plan.

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

## Data sent to cloud services

When requested, selected health data such as load, disk usage, container
names, and container health state is sent to the configured OpenAI model.
Nextcloud documents are explicitly outside the deployed milestone and must
remain opt-in in any future retrieval design.
