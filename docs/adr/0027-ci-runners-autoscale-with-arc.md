# ADR 0027: CI runners autoscale with ARC runner scale sets

- Status: Accepted
- Date: 2026-11-01
- Deployment status: applied 2026-11-01 (`k3s-bootstrap#45`–`#48`), checks and `k3s-apps` plan green on ARC; host-run AWS CLI jobs not yet run (tracker #39). Phase 2 (one global pool in a GitHub organization) open

## Context

Since 2026-08-31 the self-hosted runners were two permanent Deployments
("slots") per repository on `k3s-node-2`: 16 Pods for 8 repositories,
idle most of the time. Measured 2026-11-01: ~2.6 GiB of memory at idle
(a third of node-2) and ~4.3 GiB reserved through requests. Their limits
added up to 2485 % CPU and 387 % memory of the node, so nothing made
concurrent jobs wait, and concurrent CI could saturate the VM
(tracker #12). After every reboot all 16 started at once (tracker #29).
The SSD move (tracker #40) fixed the boot side. The idle cost and the
missing concurrency limit stayed.

Two alternatives:

- **One shared pool across repositories.** Fewer Pods, idle capacity
  shared. But GitHub only offers self-hosted runners at repository,
  organization or enterprise level, and the repositories live under the
  personal account `KandlerLi`. A shared pool needs an organization and
  moving every repository (OIDC subjects, `repo-infra`, the App,
  Renovate).
- **Autoscaling per repository** with actions-runner-controller (ARC)
  runner scale sets. No idle runner Pods, isolation per repository
  stays. It adds a controller with CRDs, a cold start per job, and the
  per-slot caches don't fit Pods that exist for one job.

## Decision

**ARC first, organization later.** `k3s-bootstrap` runs the ARC
controller (`arc-system`, `watchSingleNamespace: ci`) and one scale set
per runner repository with `minRunners: 0` and `maxRunners: 3`. Every
runner Pod requests 500m CPU and memory from the 7-day peak per Pod, so
node-2 fits about three jobs at once and further jobs wait in GitHub's
queue. Labels stay `self-hosted, home, debian`, so no workflow changed.
Runner Pods start empty (no caches); measure before adding one.

Phase 2 is still the goal: one global scale set in an organization
(`jkandler`), plus a dedicated one for `k3s-apps`, the only repository
whose runner holds a Kubernetes ServiceAccount token. AWS rights don't
argue against sharing: they are bound to the repository by the OIDC
subject, not to the runner.

## Consequences

- At idle only the controller and 8 small listeners run, no runner Pod.
- No job can leave anything behind for another, not even within one
  repository: every Pod is fresh. The cost per job, measured
  2026-11-01: ~70 s from queued to started and ~76 s of `terraform init`
  without a plugin cache (tracker #52).
- The runner image stays the community `myoung34/github-runner`, started
  with `run.sh` under ARC. The official `actions/actions-runner` lacks
  `node` and the AWS CLI that host-run jobs need (see `decisions.md`).
- Privileged dind stays confined to `ci` on node-2. The controller only
  has Roles in `ci`, and `arc-system` enforces `restricted`.
- Design details: `k3s-bootstrap`'s README, "Runner design".
