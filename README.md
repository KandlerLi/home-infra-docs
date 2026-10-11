# Home Infrastructure Documentation

Architecture and operations documentation for Julian Kandler's Debian
homeserver and its supporting automation.

This repository is the documentation source of truth for the whole
setup. The code lives in separate repositories under
[github.com/KandlerLi](https://github.com/KandlerLi);
[`docs/repositories.md`](docs/repositories.md) describes each one.

The former `home-infra-ai-context` repository was merged into this one in
2026-10 and archived; it keeps only the dated session handovers from
August 2026.

## Current status

[`docs/current-state.md`](docs/current-state.md) is the maintained record
of what is live; the C4 views under
[`docs/architecture/`](docs/architecture/README.md) show the same system as
diagrams.

## Documentation map

- [Documentation index](docs/README.md)
- [Current state](docs/current-state.md)
- [History](docs/history.md) (build and migration record)
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
