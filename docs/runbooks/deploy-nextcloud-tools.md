# Runbook: Deploy Read-Only Nextcloud Tools — retired

**Retired 2026-08-30, not applicable to the current architecture.**
This runbook described `infra/home-infra`'s own Docker/systemd-based
deployment of the `nextcloud-tools` service (`ansible/playbooks/
nextcloud-tools.yml`, `rotate-nextcloud-tools-token.yml`,
`disable-nextcloud-tools.yml`) — that whole role and its playbook
family were deleted outright once `home_agent`/`nextcloud_tools`
moved to a k3s-native Pod. None of the commands below this point still
exist; following them will fail immediately with a missing-file error,
not silently do the wrong thing.

The service itself is still live — it's just deployed differently now:
`nextcloud_tools_service.py` runs as a vendored sidecar inside
`infra/k3s-apps`' own `modules/home_agent` Deployment, alongside
`home_agent` itself, sharing a Unix socket through an `emptyDir`. It
reaches Nextcloud AIO's Apache over the k3s VM's isolated network
(`192.168.101.1:11000`) rather than loopback. There's no dedicated
deploy runbook for this shape — it's deployed the same way every other
`infra/k3s-apps` module is, through that repo's own normal Terraform
apply (CI on merge to `main`).

See `docs/home-infra-ai-context/context/current-state.md`'s own "k3s
learning cluster" section for the full cutover history and current
architecture, and `infra/k3s-apps`' own `modules/home_agent/` for the
live Terraform.

The security-boundary reasoning this runbook originally walked through
(ADR 0008's own scope, the dedicated Nextcloud account/share model)
still applies conceptually to the current k3s-native deployment — only
the specific commands here are gone, not the design.
