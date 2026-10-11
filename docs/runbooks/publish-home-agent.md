# Runbook: Publish the Home Agent — retired

**Retired 2026-09-01, not applicable to the current architecture.**
This runbook described bootstrapping [`home-infra`](https://github.com/KandlerLi/home-infra)'s own
`shared_ingress` role from scratch -- the homeserver's original,
Docker-based Traefik reverse proxy fronting Nextcloud, the agent, and
every other public route. `shared_ingress` was deleted outright once
its own replacement (k3s-native Traefik, reached via
`k3s_ingress_forward`'s DNAT relay) was confirmed carrying real
production traffic correctly -- every playbook this runbook references
(`shared-ingress-prepare.yml`, `shared-ingress.yml`,
`shared-ingress-agent.yml`, `shared-ingress-rollback.yml`) is gone,
along with the whole family of sibling publish/rollback playbooks for
every other route (`publish-deluge.yml`, `publish-grafana.yml`, etc.).

The public routing this runbook originally set up now lives entirely
in [`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' own `modules/ingress` (deployed through that
repo's normal Terraform apply), with two real changes since this
runbook was current:

- **Authentication**: Authelia forward-auth gates `agent-chain` now
  (MFA-gated OIDC SSO), not the Traefik Basic Auth this runbook
  describes generating and storing in SOPS -- SOPS itself is also
  fully retired, replaced by AWS Secrets Manager.
- **Routing shape**: `ai.jkandler.de` is no longer a single
  agent-only route -- it's split by *path* between `home_agent`
  (`/healthz`, `/v1/chat`) and `open_webui` (everything else), which
  deliberately does *not* carry Authelia forward-auth since it does
  its own app-level OIDC login instead.

See [`history.md`](../history.md)'s own "k3s
learning cluster" section for the full cutover history, and its
Authelia SSO writeup for the auth-chain details.

Everything else this runbook describes -- the original Basic Auth
rate-limiting values, the specific `community.docker` `capabilities`
vs. `cap_add` bug, the exact commit hashes -- is historical record of
a bootstrap that no longer reflects live infrastructure in any way;
kept out of this retirement note rather than repeated, since none of
it applies to the current Terraform-based routing at all.
