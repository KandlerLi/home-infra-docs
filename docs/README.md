# Documentation Index

## Baseline

- [Current state](current-state.md) — dated evidence and known gaps
- [System inventory](inventory.md) — hardware, hosts, services, and networks
- [Networking and ingress](networking.md) — DNS, ports, IPv4, and IPv6
- [Storage](storage.md) — disks, mounts, application data, and safety rules
- [Security and secrets](security-and-secrets.md) — trust boundaries and SOPS
- [Operations](operations.md) — routine Ansible and diagnostic commands
- [Glossary](glossary.md) — local terms and abbreviations
- [References](references.md) — authoritative external documentation

## C4 architecture

- [Architecture index and notation](architecture/README.md)
- [Current system context](architecture/current/system-context.md)
- [Current container view](architecture/current/container.md)
- [Infrastructure automation components](architecture/current/component-home-infra.md)
- [Home-agent components](architecture/current/component-home-agent.md)
- [Current deployment](architecture/current/deployment.md)
- [Current Ansible deployment flow](architecture/current/dynamic-ansible.md)
- [Current agent health-check flow](architecture/current/dynamic-home-agent.md)
- [Public agent routing](architecture/proposed/README.md)
- [Deployed read-only Nextcloud tools](architecture/proposed/nextcloud-tools.md)

## Decisions and runbooks

- [Architecture decision records](adr/README.md)
- [Runbooks](runbooks/README.md)

## Evidence precedence

Conflicting information should be resolved in this order:

1. Live, read-only observation of the homeserver
2. Committed files in `/home/julian/projects/infra/home-infra`
3. Uncommitted changes in that repository, marked as proposed
4. Historical handover notes, marked as historical or unverified
