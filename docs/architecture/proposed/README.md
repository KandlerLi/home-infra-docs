# Proposed Home-Agent Architecture

Status: **Proposed and not deployed** as of 2026-08-18.

The infrastructure working tree contains an implementation of the first two
agent phases, but Julian has explicitly confirmed that it has not been applied.
These diagrams document intent and the code under review, not production state.

## Views

- [System context](system-context.md)
- [Container view](container.md)
- [Component view](component.md)
- [Deployment view](deployment.md)
- [Dynamic health-check flow](dynamic-health-check.md)

## Goals

- Cloud inference rather than local LLM inference
- Explicit read-only tools
- No arbitrary shell, unrestricted SSH, or model-facing Docker socket
- Structured and sanitised tool output
- SOPS-managed API credential
- Loopback-only API until authenticated HTTPS ingress exists
- No Nextcloud document retrieval in the first milestone

## Activation prerequisites

- Review and commit the infrastructure changes.
- Add `home_agent_openai_api_key` to the encrypted SOPS file.
- Recreate the controller virtual environment at its current repository path.
- Run syntax, lint, and unit tests.
- Apply the dedicated playbook with human approval.
- Verify the local API before considering any public ingress.
