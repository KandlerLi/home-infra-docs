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
| [0003](0003-nextcloud-aio.md) | Superseded by 0033 | Operate Nextcloud through AIO without reorganising its data |
| [0004](0004-read-only-home-agent-boundary.md) | Superseded (retired 2026-11-16) | Separate a cloud-model agent from fixed host collectors |
| [0005](0005-shared-reverse-proxy.md) | Accepted | Introduce a shared authenticated HTTPS ingress layer |
| [0006](0006-local-aws-account-bootstrap.md) | Superseded by 0010 | Manage AWS deployment identities from a local-only control plane |
| [0007](0007-open-webui-over-restricted-agent.md) | Superseded (retired 2026-11-16) | Use Open WebUI only as a frontend for the restricted home-agent |
| [0008](0008-read-only-nextcloud-tool-boundary.md) | Superseded (retired 2026-11-16) | Put read-only Nextcloud access behind a dedicated bounded service |
| [0009](0009-terraform-managed-github-repository-protection.md) | Accepted | Manage GitHub repository protection through Terraform |
| [0010](0010-consolidate-repo-and-identity-control-plane.md) | Accepted | Consolidate GitHub repository and AWS identity control planes |
| [0011](0011-nextcloud-write-tools.md) | Superseded (retired 2026-11-16) | Give the home agent Nextcloud write access via a server-verified propose/confirm flow |
| [0012](0012-audio-transcription-relay.md) | Superseded (retired 2026-11-16) | Relay speech-to-text through home-agent instead of local Whisper |
| [0013](0013-remove-nextcloud-write-confirmation.md) | Superseded (retired 2026-11-16) | Drop the Nextcloud write confirmation step (amends 0011) |
| [0014](0014-nextcloud-shopping-list-tools.md) | Superseded (retired 2026-11-16) | Give the home agent Nextcloud Shopping List item tools |
| [0015](0015-sankey-export-automation.md) | Accepted | Bring the Finanzfluss Sankey exporter into Ansible and automate it |
| [0016](0016-personal-website.md) | Accepted | Host a personal website at `www.jkandler.de` on S3 + CloudFront |
| [0017](0017-monitoring-split-by-cost.md) | Accepted | Split monitoring by cost: homeserver-hosted, one AWS-side reachability check |
| [0018](0018-scoped-operator-identity.md) | Accepted | A scoped IAM identity replaces root for local Terraform applies |
| [0019](0019-homeserver-monitoring-alerting.md) | Accepted | Homeserver monitoring stack (Prometheus/Grafana/Alertmanager) and its dual-channel alerting |
| [0020](0020-blocky-dns-ad-blocking.md) | Accepted | Network-wide DNS ad-blocking with Blocky, fed into Grafana |
| [0021](0021-agent-shared-root-and-document-reading.md) | Superseded (retired 2026-11-16) | Widen the agent's Nextcloud scope to all shares; let it read PDF and xlsx |
| [0022](0022-trivy-config-scanning-and-public-repo-tradeoff.md) | Accepted | Trivy config scanning across every repo, and the public-repo branch-protection tradeoff |
| [0023](0023-paperless-ngx-document-archive.md) | Proposed | Paperless-ngx on k3s as the personal document archive |
| [0024](0024-wireguard-admin-vpn.md) | Superseded by 0025 | WireGuard on the homeserver for admin access from outside |
| [0025](0025-fritzbox-wireguard-admin-vpn.md) | Accepted | FRITZ!Box built-in WireGuard for admin access from outside |
| [0026](0026-qbittorrent-replaces-deluge.md) | Superseded (retired 2026-11-15) | qBittorrent replaces Deluge, so `apps` can enforce PodSecurity `restricted` |
| [0027](0027-ci-runners-autoscale-with-arc.md) | Accepted | CI runners autoscale with ARC runner scale sets |
| [0028](0028-vaultwarden-public-without-forward-auth.md) | Accepted | Vaultwarden on k3s, its client API public without Authelia forward-auth |
| [0029](0029-vaultwarden-login-through-authelia.md) | Accepted | Vaultwarden logs in through Authelia only (SSO), amends 0028 |
| [0030](0030-credentials-outside-secrets-manager.md) | Accepted | Credentials that can't live in Secrets Manager go into `pass` or onto paper |
| [0031](0031-argocd-ui-behind-authelia.md) | Accepted | Argo CD's UI public on `argo.jkandler.de`, behind Authelia forward-auth and OIDC |
| [0032](0032-cluster-secrets-through-external-secrets.md) | Accepted | Cluster Secrets through the External Secrets Operator, authenticated by OIDC federation |
| [0033](0033-nextcloud-on-k3s.md) | Accepted | Move Nextcloud from AIO on the host to plain manifests on k3s |
| [0034](0034-eurooffice-on-k3s.md) | Accepted | Euro-Office on k3s, rootless, behind forward-auth |
| [0035](0035-monitoring-off-the-host.md) | Proposed | Monitoring off the host: Prometheus, Loki and blackbox to k3s, no Docker on the host |
| [0036](0036-minecraft-server-on-k3s.md) | Accepted (own-node amendment Proposed) | Minecraft server on k3s, public on 25565/tcp and 19132/udp+tcp |
| [0037](0037-k3s-bootstrap-through-platform-argocd.md) | Accepted | `k3s-bootstrap` as GitOps manifests, synced by a separate platform Argo CD |
