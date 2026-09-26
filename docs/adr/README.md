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
| [0006](0006-local-aws-account-bootstrap.md) | Superseded by 0010 | Manage AWS deployment identities from a local-only control plane |
| [0007](0007-open-webui-over-restricted-agent.md) | Accepted | Use Open WebUI only as a frontend for the restricted home-agent |
| [0008](0008-read-only-nextcloud-tool-boundary.md) | Accepted | Put read-only Nextcloud access behind a dedicated bounded service |
| [0009](0009-terraform-managed-github-repository-protection.md) | Accepted | Manage GitHub repository protection through Terraform |
| [0010](0010-consolidate-repo-and-identity-control-plane.md) | Accepted | Consolidate GitHub repository and AWS identity control planes |
| [0011](0011-nextcloud-write-tools.md) | Accepted | Give the home agent Nextcloud write access via a server-verified propose/confirm flow |
| [0012](0012-audio-transcription-relay.md) | Accepted | Relay speech-to-text through home-agent instead of local Whisper |
| [0013](0013-remove-nextcloud-write-confirmation.md) | Accepted | Drop the Nextcloud write confirmation step (amends 0011) |
| [0014](0014-nextcloud-shopping-list-tools.md) | Accepted | Give the home agent Nextcloud Shopping List item tools |
| [0015](0015-sankey-export-automation.md) | Accepted | Bring the Finanzfluss Sankey exporter into Ansible and automate it |
| [0016](0016-personal-website.md) | Accepted | Host a personal website at `www.jkandler.de` on S3 + CloudFront |
| [0017](0017-monitoring-split-by-cost.md) | Accepted | Split monitoring by cost: homeserver-hosted, one AWS-side reachability check |
| [0018](0018-scoped-operator-identity.md) | Accepted | A scoped IAM identity replaces root for local Terraform applies |
| [0019](0019-homeserver-monitoring-alerting.md) | Accepted | Homeserver monitoring stack (Prometheus/Grafana/Alertmanager) and its dual-channel alerting |
| [0020](0020-blocky-dns-ad-blocking.md) | Accepted | Network-wide DNS ad-blocking with Blocky, fed into Grafana |
| [0021](0021-agent-shared-root-and-document-reading.md) | Accepted | Widen the agent's Nextcloud scope to all shares; let it read PDF and xlsx |
| [0022](0022-trivy-config-scanning-and-public-repo-tradeoff.md) | Accepted | Trivy config scanning across every repo, and the public-repo branch-protection tradeoff |
| [0023](0023-paperless-ngx-document-archive.md) | Proposed | Paperless-ngx on k3s as the personal document archive |
| [0024](0024-wireguard-admin-vpn.md) | Superseded by 0025 | WireGuard on the homeserver for admin access from outside |
| [0025](0025-fritzbox-wireguard-admin-vpn.md) | Accepted | FRITZ!Box built-in WireGuard for admin access from outside |
