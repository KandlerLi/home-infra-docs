# Completed Public Home-Agent Routing

Status: **Deployed and verified** as of 2026-08-19. This directory retains the
original proposal path for link stability, but the architecture is current.

The private agent is deployed and verified on `127.0.0.1:8090`. This section
documents the authenticated publication of `ai.jkandler.de`.

## Views

- [Deployment view](deployment.md)

## Goals

- Preserve the verified Nextcloud route through pinned Traefik `v3.7.1`.
- Terminate HTTPS for the new hostname using automatic ACME certificates.
- Authenticate agent requests independently of model behaviour.
- Apply request-size, rate, and upstream timeout limits.
- Publish no AAAA record until IPv6 routing and TLS are verified end to end.

## Deployed outcome

- Nextcloud remains healthy behind the shared proxy.
- Agent authentication, rate limits, request-size limits, and headers are active.
- `ai.jkandler.de` resolves through the Terraform-managed CNAME without AAAA.
- Unauthenticated requests return 401; authenticated health and chat return 200.
