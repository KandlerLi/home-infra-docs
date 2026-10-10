# Architecture Summary

## Public request paths

```text
Remote Nextcloud client
  -> Route 53 CNAME/A
  -> FRITZ!Box NAT
  -> Traefik :443
  -> 127.0.0.1:11000
  -> Nextcloud AIO Apache and application stack
```

```text
Authenticated agent client
  -> ai.jkandler.de CNAME/A
  -> FRITZ!Box NAT
  -> Traefik :443
     - ACME TLS
     - Basic Auth
     - 10 requests/minute, burst 5
     - 16 KiB request-body limit
     - security headers
  -> 127.0.0.1:8090
  -> home-agent
  -> Anthropic Messages API
  -> approved home-tools calls over Unix socket
  -> fixed read-only host collectors
```

## DNS update path

```text
FRITZ!Box
  -> authenticated AWS API Gateway endpoint
  -> Python Lambda
  -> Route 53 jkandler.de A record

Terraform/GitHub Actions
  -> stable nextcloud and ai CNAME records
```

## AWS automation trust path

Current deployed ownership is:

```text
Trusted local operator with short-lived bootstrap identity + GitHub token
  -> local-only repo-infra Terraform root
  -> account-wide GitHub OIDC provider
  -> repository-bound DynDNS plan/apply roles and policies
  -> GitHub repository settings (dyndns, testing)

GitHub Actions OIDC token
  -> plan or protected-environment apply role
  -> dyndns Terraform state and allowed workload resources only
```

Neither DynDNS automation role may access the `repo-infra` state key or modify
its own trust or permissions. `repo-infra` (renamed from `github-infra`)
absorbed this trust path from the retired `aws-account-bootstrap` root on
2026-08-21.

## Agent trust boundary

The cloud model chooses which declared read-only tool it needs. Trusted local
code decides how that tool is fulfilled. The model cannot supply a shell
command, path, unit name, or container name.

The `home-agent` container sees only:

- a read-only API-key file
- the protected home-tools Unix-socket directory
- its own application files and bounded runtime state

The host-side service owns Docker inspection and returns a sanitised subset. It
does not return container environments or arbitrary logs.

## Browser-chat path

Deployed:

```text
Browser
  -> ai.jkandler.de / Traefik
     - Open WebUI route: TLS, rate/body limits, security headers
     - exact /healthz and /v1/chat: existing Basic Auth and API limits
  -> 127.0.0.1:8091 Open WebUI with its own login
  -> non-masqueraded home-agent-frontend Docker bridge
  -> home-agent OpenAI-compatible endpoint
  -> current Anthropic Messages provider and fixed home-tools boundary
```

Open WebUI has no real provider key or host access. HTTP Basic Auth is not used
on its catch-all route because it conflicts with Open WebUI's Bearer
`Authorization` header. The first administrator must be created and signup
confirmed disabled over an SSH tunnel before the public route is switched.
Private onboarding and the guarded route switch completed on 2026-08-19.
Authenticated public chat was subsequently user-confirmed, and the deployed
inventory flags were persisted.

## Technology choices

- configuration: Ansible for the host; Terraform for AWS/DNS; local-only
  Terraform for AWS automation identities
- runtime: Docker for AIO, Traefik, home-agent, and Open WebUI;
  systemd for home-tools
- inference: Anthropic cloud API, currently `claude-opus-5` at `low` output
  effort (switched from OpenAI `gpt-5.4-mini` 2026-09-18); OpenAI is still used
  for Whisper speech-to-text only
- secrets: AWS Secrets Manager for everything (the 2026-09-09
  SOPS-to-Secrets-Manager cutover retired SOPS/GPG entirely)
- ingress: pinned Traefik `v3.7.1`, host network, file provider, no Docker socket
- documentation: C4 views in `home-infra-docs`

See `/home/julian/projects/docs/home-infra-docs/docs/architecture/README.md` for the
authoritative diagrams and notation.

## Prepared Nextcloud read path

Not deployed:

```text
home-agent
  -> protected /run/nextcloud-tools Unix socket
  -> dedicated nextcloud-tools systemd service
  -> PROPFIND or bounded text GET to 127.0.0.1:11000
  -> dedicated non-admin Nextcloud account
  -> read-only AI Workspace share
```

The service account, validated credential, host service, agent socket, and
owner-granted share are deployed. Health passes, but the first bounded list
returned 503; diagnostic reapply then exposed a socket restart race. Revision
`cb6a3d2` preserves the systemd runtime directory across restarts and forces a
container remount during dedicated plays and is deployed. The logical share is
`AI Workspace`, but Nextcloud mounts it under `/Shared`, so the bounded WebDAV
root is `Shared/AI Workspace`. Open WebUI never receives the socket or
credential. Writes, bulk indexing, binary extraction, and image-content
analysis remain outside this path. Direct list/search/small-text-read checks and
Open WebUI acceptance pass; desired-state flags are persisted.
