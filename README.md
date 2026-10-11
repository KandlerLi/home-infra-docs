# Home Infrastructure Documentation

Architecture and operations documentation for Julian Kandler's Debian
homeserver and its supporting automation.

This repository is the documentation source of truth. Infrastructure code is
kept separately in:

```text
/home/julian/projects/infra/home-infra
```

Agent working rules live in the workspace `CLAUDE.md` and
`.claude/skills/` (`/home/julian/projects`). The former
`home-infra-ai-context` repository was merged into this one in 2026-10 and
archived; it keeps only the dated session handovers from August 2026.

The central Terraform state-bucket bootstrap project is:

```text
/home/julian/projects/bootstrap/terraform-state
```

GitHub repository configuration and the account-wide AWS identity/GitHub
OIDC control plane are both managed from:

```text
/home/julian/projects/github/repo-infra
```

Both bootstrap projects are local Git repositories whose tracked sources
exclude all Terraform state artifacts. Since 2026-08-21 every project listed
above lives one level under a parent folder that groups it by purpose:
`/home/julian/projects/docs/`, `/home/julian/projects/bootstrap/`, or
`/home/julian/projects/infra/`. The complete inventory boundary is every
project directory nested one level under one of those three folders, not
only Git roots.

## Current status

[`docs/current-state.md`](docs/current-state.md) is the maintained,
dated record of what is live. The C4 pages under `docs/architecture/current/`
date from August 2026 (Docker era); [`docs/architecture/overview.md`](docs/architecture/overview.md)
is the up-to-date architecture summary until they are redrawn.

## Documentation map

- [Documentation index](docs/README.md)
- [Current state](docs/current-state.md)
- [History](docs/history.md) (build and migration record)
- [System inventory](docs/inventory.md)
- [Architecture overview](docs/architecture/overview.md)
- [C4 architecture index](docs/architecture/README.md)
- [Repositories](docs/repositories.md)
- [Repository security model](docs/repository-security.md)
- [Networking and ingress](docs/networking.md)
- [Storage](docs/storage.md)
- [Security and secrets](docs/security-and-secrets.md)
- [Operations](docs/operations.md)
- [Architecture decisions](docs/adr/README.md)
- [Decisions log](docs/decisions.md) (smaller decisions without their own ADR)
- [Runbooks](docs/runbooks/README.md)
- [Glossary](docs/glossary.md)
- [References](docs/references.md)

## Documentation principles

- Clearly separate deployed state from proposed state.
- Treat the infrastructure repository and live observations as evidence.
- Never copy secret values, private keys, tokens, document contents, or
  container environment variables into this repository.
- Date runtime observations because they become stale.
- Update architecture diagrams in the same change as architecture decisions.
- Keep diagrams understandable without relying on colour alone.

See [CONTRIBUTING.md](CONTRIBUTING.md) for the update workflow.
