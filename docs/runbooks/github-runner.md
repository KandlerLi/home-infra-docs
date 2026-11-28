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

**Since 2026-11-01 the runners are ARC scale sets**, one per repository
with one Pod per job ([ADR 0027](../adr/0027-ci-runners-autoscale-with-arc.md),
[`k3s-bootstrap`'s docs](https://github.com/KandlerLi/k3s-bootstrap/blob/main/docs/README.md), "Runner design"); the per-repository
Deployments described above are gone.

Since 2026-11-28 ARC is part of [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)'s `platform/apps/arc/`,
synced by hand by the platform Argo CD (ADR 0037). Only the GitHub App
Secret `arc-github-app` stays a local Terraform apply.

## Adding a repository

1. Grant the GitHub App (not a PAT) access to the new repository, via
   the App's own installation settings.
2. Add `runner: true` to the repository's entry in
   [`repo-infra/config.yml`](https://github.com/KandlerLi/repo-infra/blob/main/config.yml).
3. In [`home-infra`](https://github.com/KandlerLi/home-infra), run
   `.venv/bin/python scripts/sync_github_runner_repositories.py`. It
   writes the sibling `k3s-bootstrap`'s
   `platform/apps/arc/scale-sets/kustomization.yaml` and an overlay for
   the new repository (an existing overlay is never rewritten). Review the
   diff and open the PR in `k3s-bootstrap`, not `home-infra`.
4. After the merge, sync `platform-root` and then the new
   `arc-runners-<repo>` by hand on `argo-platform.jkandler.de`.
5. Confirm the new scale set's listener is running
   (`kubectl get pods -n arc-system`) and the scale set shows under the
   repository's runner settings. No runner is listed until a job runs.

To remove a repository: remove `runner: true` from `config.yml`, re-run
the sync script, merge, and sync `platform-root`. The platform Argo CD
never prunes, so then delete the orphaned Application
`arc-runners-<repo>` in its UI (with its resources); ARC deletes the scale
set from GitHub.
