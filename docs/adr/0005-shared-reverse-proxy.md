# ADR 0005: Introduce a Shared Authenticated HTTPS Ingress Layer

- Status: Accepted and deployed
- Date: 2026-08-18
- Deployment date: 2026-08-19

## Context

Nextcloud AIO originally owned public port 443. The deployed agent uses a
loopback-only HTTP API. Multiple hostnames on one public address need hostname
routing, TLS termination, and independent authentication.

## Decision

Before publishing `ai.jkandler.de`, use an Ansible-managed shared reverse
proxy using the pinned Traefik `v3.7.1` container and file provider. Do not
mount the Docker socket. Move Nextcloud AIO to its supported reverse-proxy
mode at `127.0.0.1:11000`, preserve `nextcloud.jkandler.de`, and protect the
agent with Basic Auth, rate limits, a 16 KiB request-body limit, and security
headers.

## Consequences

- The change affects the most important production ingress path and requires a
  maintenance window and rollback plan.
- Port 443 moves from AIO Apache to the shared proxy.
- Traefik uses host networking so it can reach only loopback-bound upstreams;
  it drops all capabilities except `NET_BIND_SERVICE` and uses a read-only
  root filesystem.
- Routing is defined in files rather than Docker labels, so Traefik does not
  receive the Docker socket.
- `ai.jkandler.de` DNS is created only after local routing and authentication
  work.
- No AAAA record is added without verified IPv6 routing.
