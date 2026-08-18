# ADR 0004: Separate the AI Agent from Fixed Host Collectors

- Status: Proposed
- Date: 2026-08-18

## Context

The homeserver has insufficient resources for a useful primary local LLM. A
cloud model can interpret server state, but it must not receive unrestricted
root, SSH, shell, Docker, or secret access.

## Decision

Use a small model-facing `home-agent` container and a separate host-side
`home-tools` service. Expose only argument-free, read-only, structured tools
over a group-restricted Unix socket. Keep the Docker socket out of the agent
container.

## Consequences

- Trusted local code controls how observations are collected and sanitised.
- The host collector remains privileged in the narrow sense that Docker group
  membership can access the Docker API; systemd sandboxing and a fixed code
  path reduce the exposed surface.
- The agent can report but cannot remediate.
- Selected host data leaves the server when sent to the cloud model.
- This decision remains proposed until the implementation is reviewed and
  deployed.
