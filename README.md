# Home Infrastructure Documentation

Architecture and operations documentation for Julian Kandler's Debian
homeserver and its supporting automation.

This repository is the documentation source of truth. Infrastructure code is
kept separately in:

```text
/home/julian/projects/home-infra
```

## Snapshot status

The initial documentation snapshot was taken on **2026-08-18**.

| Label | Meaning |
| --- | --- |
| Current | Observed live or confirmed in committed infrastructure |
| Configured | Defined in committed Ansible; runtime may be unverified |
| Proposed | Designed or implemented locally, but not deployed |
| Unknown | Observed state whose owner or purpose has not yet been confirmed |

The private `home-agent` implementation is **deployed and verified** on the
homeserver. Its HTTP API is reachable only through `127.0.0.1:8090`; it is not
publicly exposed. `ai.jkandler.de` has no DNS record and no shared reverse
proxy is configured.

## Documentation map

- [Documentation index](docs/README.md)
- [Current state](docs/current-state.md)
- [System inventory](docs/inventory.md)
- [C4 architecture index](docs/architecture/README.md)
- [Networking and ingress](docs/networking.md)
- [Storage](docs/storage.md)
- [Security and secrets](docs/security-and-secrets.md)
- [Operations](docs/operations.md)
- [Architecture decisions](docs/adr/README.md)
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
