# Home Infrastructure Documentation

Architecture and operations documentation for Julian Kandler's Debian
homeserver and its supporting automation.

This repository is the documentation source of truth. Infrastructure code is
kept separately in:

```text
/home/julian/projects/home-infra
```

Secret-free cross-chat and cross-model handover context is kept separately in:

```text
/home/julian/projects/home-infra-ai-context
```

The central Terraform state-bucket bootstrap project is:

```text
/home/julian/projects/terraform-state
```

The local-only AWS identity and GitHub OIDC control plane is:

```text
/home/julian/projects/aws-account-bootstrap
```

Both bootstrap projects are local Git repositories whose tracked sources
exclude all Terraform state artifacts. The complete inventory boundary is
every top-level directory under `/home/julian/projects`, not only Git roots.

## Snapshot status

The initial documentation snapshot was taken on **2026-08-18** and the shared
ingress deployment was recorded on **2026-08-19**.

| Label | Meaning |
| --- | --- |
| Current | Observed live or confirmed in committed infrastructure |
| Configured | Defined in committed Ansible; runtime may be unverified |
| Proposed | Designed or implemented locally, but not deployed |
| Unknown | Observed state whose owner or purpose has not yet been confirmed |

The `home-agent` implementation is **deployed and verified**. Its container API
remains loopback-only at `127.0.0.1:8090`; pinned Traefik `v3.7.1` publishes it
at `ai.jkandler.de` behind Basic Auth and request limits. Traefik also routes
public Nextcloud traffic to AIO Apache on `127.0.0.1:11000`.

Open WebUI is **deployed and publicly routed** at `ai.jkandler.de`. It is
healthy on `127.0.0.1:8091`; exact `/healthz` and `/v1/chat` routes continue to
use the existing Basic-Auth-protected JSON agent API. Administrator onboarding
is complete and anonymous signup is disabled. The frontend has no real provider
credential or host access, and Nextcloud data access remains a later separately
reviewed milestone. Authenticated public login/chat is user-confirmed and the
deployed enable flags are persisted in inventory.

The read-only Nextcloud tool boundary and `AI Workspace` share are **deployed
privately**, but functional validation is pending. The service/socket health
checks initially passed and the share is mounted for `home-agent`. After the
runtime-directory fix, the container can reach the socket reliably. The bounded
list still returns 503, while this host's service journal yields no diagnostic
records. Revision `8ceb715` returns only an allowlisted status/transport category
through the protected socket and awaits reapply. No persistent inventory enable
flag exists yet.

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
