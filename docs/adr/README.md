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
| [0005](0005-shared-reverse-proxy.md) | Proposed | Introduce a shared authenticated HTTPS ingress layer |
