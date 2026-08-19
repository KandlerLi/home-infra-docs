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

## Public exposure requirements

The agent API is not ready for public exposure. Before publishing
`ai.jkandler.de`, require at least:

- HTTPS termination
- authentication independent of the model prompt
- rate and request-size limits
- explicit proxy trust configuration
- security headers appropriate to the UI/API
- audit-friendly logs that exclude prompts, secret values, and tool payloads
- a rollback plan for the Nextcloud ingress migration

## Data sent to cloud services

When requested, selected health data such as load, disk usage, container
names, and container health state is sent to the configured OpenAI model.
Nextcloud documents are explicitly outside the deployed milestone and must
remain opt-in in any future retrieval design.
