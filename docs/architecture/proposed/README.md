# Proposed and Retained Architecture Views

The public home-agent routing view is **deployed and verified** as of
2026-08-19 and remains here for link stability. The deployed AWS
account-bootstrap view has moved to `../current/account-bootstrap.md`.

The private agent is deployed and verified on `127.0.0.1:8090`. This section
retains its publication view and documents the prepared, not-yet-deployed
Open WebUI frontend for `ai.jkandler.de`.

## Views

- [Deployment view](deployment.md)
- [Open WebUI frontend](open-webui.md)

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

## Prepared next step

- Deploy Open WebUI on host loopback port 8091.
- Connect it only to the restricted home-agent over an internal Docker network.
- Bootstrap and validate the first UI administrator over an SSH tunnel.
- Replace only the hostname catch-all route after an explicit publish confirmation.
- Preserve the exact `/healthz` and `/v1/chat` API routes for compatibility.
