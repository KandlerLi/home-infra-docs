# Architecture Decision Records

Architecture decisions use short Markdown records with one of these states:

- **Accepted** — describes the current intended architecture
- **Proposed** — not yet approved or deployed
- **Superseded** — retained for history but replaced by another decision

## Index

| ADR | Status | Decision |
| --- | --- | --- |
| [0001](0001-ansible-as-infrastructure-source-of-truth.md) | Accepted | Use Ansible and Git as the infrastructure source of truth |
| [0002](0002-sops-with-gpg-for-secrets.md) | Accepted | Encrypt committed secrets with SOPS and GPG |
| [0003](0003-nextcloud-aio.md) | Accepted | Operate Nextcloud through AIO without reorganising its data |
| [0004](0004-read-only-home-agent-boundary.md) | Accepted | Separate a cloud-model agent from fixed host collectors |
| [0005](0005-shared-reverse-proxy.md) | Accepted | Introduce a shared authenticated HTTPS ingress layer |
| [0006](0006-local-aws-account-bootstrap.md) | Accepted | Manage AWS deployment identities from a local-only control plane |
| [0007](0007-open-webui-over-restricted-agent.md) | Accepted | Use Open WebUI only as a frontend for the restricted home-agent |
| [0008](0008-read-only-nextcloud-tool-boundary.md) | Accepted | Put read-only Nextcloud access behind a dedicated bounded service |
