# Runbook: Enable the Nextcloud Shopping List tools — retired

**Retired 2026-08-30, not applicable to the current architecture.**
This runbook's own deploy step (`ansible/playbooks/nextcloud-tools.yml`
and `home-agent.yml`) belonged to [`home-infra`](https://github.com/KandlerLi/home-infra)'s Docker/systemd-based
`nextcloud-tools` deployment — that whole role and its playbook family
were deleted outright once `home_agent`/`nextcloud_tools` moved to a
k3s-native Pod ([`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' own `modules/home_agent`). The
commands below this point no longer exist.

The feature this runbook describes (ADR 0014's Shopping List tools,
add/check/uncheck/remove, executing immediately per ADR 0013) is
still **implemented in code but not yet deployed** — same status as
before this retirement, just via a different, current mechanism:
enabling it now means deploying [`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' own
`modules/home_agent` through that repo's normal Terraform apply, not
these dead Ansible plays. See
[`current-state.md`](../current-state.md) and the private task list for the
current status.

The two real, still-applicable pieces of this runbook's original
content:

- **Step 1's manual app/list/share setup** (installing the Shopping
  List app, creating a list, sharing it with **edit** permission to
  the `home-agent` account) is still the real prerequisite, unrelated
  to which deployment mechanism serves the request.
- **Step 4's live end-to-end verification checklist** (add, check,
  uncheck, remove, ambiguous/unknown-item handling) is still the right
  way to validate this feature once it's actually deployed via the
  current mechanism — the test steps themselves didn't change, only
  how you get the code running.
