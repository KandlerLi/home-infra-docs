# Runbook: Enable Nextcloud Write Tools — retired

**Retired 2026-08-30, not applicable to the current architecture.**
This runbook's own deploy step (`ansible/playbooks/nextcloud-tools.yml`
and `home-agent.yml`) belonged to `infra/home-infra`'s Docker/systemd-based
`nextcloud-tools` deployment — that whole role and its playbook family
were deleted outright once `home_agent`/`nextcloud_tools` moved to a
k3s-native Pod (`infra/k3s-apps`' own `modules/home_agent`). The
commands below this point no longer exist.

The feature this runbook describes (ADR 0011's write capability,
create/update/delete/move, executing immediately per ADR 0013) is
still **implemented in code but not yet deployed** — same status as
before this retirement, just via a different, current mechanism:
enabling it now means deploying `infra/k3s-apps`' own
`modules/home_agent` through that repo's normal Terraform apply, not
these dead Ansible plays. See
`docs/home-infra-ai-context/context/current-state.md`'s "Known gaps"
section for the current, authoritative status, and its "k3s learning
cluster" section for the cutover history.

The two real, still-applicable pieces of this runbook's original
content:

- **Step 1's manual share-widening** ("Can edit" on the `home-agent`
  share) is still the real prerequisite, unrelated to which deployment
  mechanism serves the request.
- **Step 4's live end-to-end verification checklist** (create, update,
  conflict handling, folder-delete refusal, move/rename) is still the
  right way to validate this feature once it's actually deployed via
  the current mechanism — the test steps themselves didn't change,
  only how you get the code running.
