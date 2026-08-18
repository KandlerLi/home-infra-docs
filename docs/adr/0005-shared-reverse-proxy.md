# ADR 0005: Introduce a Shared Authenticated HTTPS Ingress Layer

- Status: Proposed
- Date: 2026-08-18

## Context

Nextcloud AIO directly owns public port 443. The proposed agent uses a
loopback-only HTTP API. Multiple hostnames on one public address need hostname
routing, TLS termination, and independent authentication.

## Decision

Before publishing `ai.jkandler.de`, introduce an Ansible-managed shared reverse
proxy. Move Nextcloud AIO to its supported reverse-proxy mode, preserve the
existing Nextcloud hostname, and protect the agent with authentication and
rate limits.

## Consequences

- The change affects the most important production ingress path and requires a
  maintenance window and rollback plan.
- Port 443 moves from AIO Apache to the shared proxy.
- `ai.jkandler.de` DNS is created only after local routing and authentication
  work.
- No AAAA record is added without verified IPv6 routing.
