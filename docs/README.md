# Documentation Index

## Baseline

- [Current state](current-state.md) — what is live now
- [History](history.md) — build and migration record (not maintained; code comments point here)
- [Networking and ingress](networking.md) — DNS, ports, IPv4, and IPv6
- [Storage](storage.md) — disks, mounts, application data, and safety rules
- [Security and secrets](security-and-secrets.md) — trust boundaries and Secrets Manager
- [Repositories](repositories.md) — every repo, its role, CI and validation
- [Repository security model](repository-security.md) — governance, OIDC, plan/apply split
- [Operations](operations.md) — routine Ansible and diagnostic commands
- [Glossary](glossary.md) — local terms and abbreviations
- [References](references.md) — authoritative external documentation

## C4 architecture

- [Architecture overview](architecture/overview.md) — prose summary
- [Architecture index and notation](architecture/README.md) — all C4 views

## Decisions and runbooks

- [Architecture decision records](adr/README.md)
- [Decisions log](decisions.md) — smaller decisions without their own ADR
- [Runbooks](runbooks/README.md)

## Evidence precedence

Conflicting information should be resolved in this order:

1. Live, read-only observation of the homeserver
2. Committed files in [`home-infra`](https://github.com/KandlerLi/home-infra)
3. Uncommitted changes in that repository, marked as proposed
4. Historical handover notes (archived `home-infra-ai-context` repo), marked
   as historical or unverified
