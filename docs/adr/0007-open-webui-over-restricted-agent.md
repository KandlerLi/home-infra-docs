# ADR 0007: Use Open WebUI over the restricted home-agent

- Status: Accepted
- Date: 2026-08-19

## Context

`ai.jkandler.de` currently exposes a bounded JSON agent API but no browser chat
window. The desired UI should use OpenAI now, permit a later provider or local
model change, and eventually support deliberately scoped Nextcloud workflows.
Giving a general web UI the OpenAI key, Docker socket, host filesystem, or
Nextcloud storage would bypass the existing safety boundary.

## Decision

Use Open WebUI as an authenticated chat and conversation frontend only. It
connects to a bounded OpenAI-compatible API implemented by `home-agent` over an
non-masqueraded Docker bridge. `home-agent` continues to hold the real provider
key, fixed server-side instructions, tool registry, and request/tool limits.

Pin Open WebUI by release and image digest. Publish it on host loopback first,
bootstrap the initial administrator privately, and use a separately confirmed
Traefik route switch. Use Open WebUI authentication on the UI route because its
Bearer token cannot coexist with HTTP Basic Auth in one `Authorization` header.
Retain Basic Auth on the direct health/chat API routes. Disable optional Open
WebUI features that widen the execution or data boundary.

Do not connect Nextcloud in this milestone. Introduce future Nextcloud access
as a separate least-privilege tool service with narrow credentials and explicit
write confirmation.

The private runtime was deployed and validated on 2026-08-19. Public routing
remains a separate guarded step after private administrator onboarding.

## Consequences

- Users gain a normal chat window and stored conversations without relocating
  the cloud-provider credential.
- The provider can later change behind `home-agent`; a local model will require
  a reviewed provider change rather than a frontend redesign.
- Open WebUI adds an application login and new sensitive state that requires
  backup and upgrade procedures.
- The upstream image currently needs a writable root filesystem to rewrite
  bundled static assets. Other container restrictions mitigate but do not
  remove that residual risk.
- Open WebUI is a larger dependency than the existing API and must not receive
  plugins, uploads, code execution, or host mounts by default.
