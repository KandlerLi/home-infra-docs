# Runbook: Operate the GitHub Runner VM — retired

**Retired 2026-08-31, not applicable to the current architecture.**
This runbook described a dedicated libvirt VM (`github-runner`,
`192.168.100.10` on `virbr10`) running a single self-hosted GitHub
Actions runner for one repository at a time. The whole VM was
decommissioned and [`home-infra`](https://github.com/KandlerLi/home-infra)'s own `ansible/roles/github_runner`
deleted outright once the runner architecture moved to k3s: one
Deployment per repository ([`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)'s own
`modules/github_runner`), pinned to a dedicated, tainted node
(`k3s-node-2`, `ci=github-runner:NoSchedule`) so CI workloads never
compete with or endanger anything on `k3s-node-1`. Every repository in
[`repo-infra/config.yml`](https://github.com/KandlerLi/repo-infra/blob/main/config.yml)'s `runner: true` list gets its own
runner now, not just `dyndns`.

**Authentication also changed since this runbook was current**:
runner registration used a shared, manually-rotated fine-grained PAT
until 2026-09-16, when it switched to GitHub App installation tokens
(`kandlerli-home-infra-runner`) — `myoung34/github-runner` (the same
community image, now running in k3s instead of a VM) mints its own
short-lived registration token internally. See the private task list's
"GitHub App installation tokens" entry for that history.

There's no dedicated operate/apply runbook for the current shape —
it's applied the same way [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)'s other rare
changes are, a local `terraform apply` with the k3s node's own
cluster-admin kubeconfig (human-apply-only, never CI — see that
repo's own README).

## Adding a repository — still the same mechanism, different apply step

This runbook's original "Adding a repository" workflow is still
substantially correct, just with a different final step. Confirmed
directly against [`home-infra`](https://github.com/KandlerLi/home-infra)'s own
`scripts/sync_github_runner_repositories.py` (its own docstring notes
it used to also generate an Ansible variable file for the now-deleted
VM-based role; Terraform is its only remaining consumer):

1. Grant the GitHub App (not a PAT) access to the new repository, via
   the App's own installation settings.
2. Add `runner: true` to the repository's entry in
   [`repo-infra/config.yml`](https://github.com/KandlerLi/repo-infra/blob/main/config.yml).
3. In [`home-infra`](https://github.com/KandlerLi/home-infra), run
   `.venv/bin/python scripts/sync_github_runner_repositories.py` --
   this writes directly to the sibling
   [`k3s-bootstrap/repositories.auto.tfvars.json`](https://github.com/KandlerLi/k3s-bootstrap/blob/main/repositories.auto.tfvars.json) (Terraform
   auto-loads any `*.auto.tfvars.json` in its root module directory,
   nothing needs to reference it explicitly). Review the diff there
   and commit it in `k3s-bootstrap`, not `home-infra`.
4. Apply [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap) (human-apply-only, never CI).
5. Confirm the new runner is online in the repository settings.

Removing configuration does not automatically unregister or delete a
runner. Decommission it explicitly first (scale its Deployment to
zero or remove its `for_each` entry), then remove `runner: true` from
`config.yml` and re-run the sync script.
