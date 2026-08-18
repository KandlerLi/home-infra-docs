# ADR 0003: Operate Nextcloud through All-in-One

- Status: Accepted
- Date: 2026-08-18

## Context

Nextcloud is a production service with important existing data on the large
HDD. Replacing or reorganising the installation introduces unnecessary risk.

## Decision

Manage the Nextcloud AIO master container with Ansible and allow AIO to manage
its sibling containers and volumes. Keep the configured data directory below
`/mnt/black-hdd`.

## Consequences

- AIO owns application lifecycle details and requires Docker socket access.
- The master configuration volume and sibling volumes are recovery-critical.
- Storage changes require explicit migration planning.
- A future shared reverse proxy must follow the supported AIO proxy mode.
