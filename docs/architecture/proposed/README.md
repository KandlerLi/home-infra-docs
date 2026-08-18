# Proposed Public Home-Agent Ingress

Status: **Proposed and not deployed** as of 2026-08-18.

The private agent is deployed and verified on `127.0.0.1:8090`. This section
documents only the remaining public-ingress design for `ai.jkandler.de`.

## Views

- [Deployment view](deployment.md)

## Goals

- Preserve the existing Nextcloud service during ingress migration.
- Terminate HTTPS for both existing and new hostnames through a shared proxy.
- Authenticate agent requests independently of model behaviour.
- Apply request-size, rate, and upstream timeout limits.
- Publish no AAAA record until IPv6 routing and TLS are verified end to end.

## Publication prerequisites

- Select and configure an Ansible-managed shared reverse proxy.
- Prepare Nextcloud AIO's supported reverse-proxy mode and rollback plan.
- Validate the existing Nextcloud hostname before adding agent routing.
- Add authentication and rate limits for the agent upstream.
- Test locally and externally before creating `ai.jkandler.de` DNS.
