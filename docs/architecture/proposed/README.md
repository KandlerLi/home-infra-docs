# Remaining Public Home-Agent Routing

Status: **Partially deployed** as of 2026-08-19. Shared Traefik ingress and the
Nextcloud route are current; the agent route and DNS name remain proposed.

The private agent is deployed and verified on `127.0.0.1:8090`. This section
documents only the remaining authenticated publication of `ai.jkandler.de`.

## Views

- [Deployment view](deployment.md)

## Goals

- Preserve the verified Nextcloud route through pinned Traefik `v3.7.1`.
- Terminate HTTPS for the new hostname using automatic ACME certificates.
- Authenticate agent requests independently of model behaviour.
- Apply request-size, rate, and upstream timeout limits.
- Publish no AAAA record until IPv6 routing and TLS are verified end to end.

## Publication prerequisites

- Keep the deployed Nextcloud route healthy while adding agent routing.
- Add authentication and rate limits for the agent upstream.
- Test locally and externally before creating `ai.jkandler.de` DNS.
