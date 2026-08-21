# ADR 0009: Manage GitHub repository protection through Terraform

- Status: Accepted
- Date: 2026-08-20
- Deployment status: Applied

**2026-08-21 update**: `github-infra` was renamed to
`/home/julian/projects/repo-infra` and absorbed the AWS identity control
plane described in ADR 0006. The decisions and consequences below remain
accurate; only the repository name/path changed. See
[ADR 0010](0010-consolidate-repo-and-identity-control-plane.md).

## Context

`KandlerLi/dyndns` was hardened by hand and by a repository-specific script
(`dyndns/scripts/protect-repository.sh`), documented as a reusable pattern in
`home-infra-ai-context/context/repository-security-blueprint.md`. Two GitHub
Actions settings were never captured by that script or by any Terraform
resource: whether third-party Actions must be pinned by commit SHA
(`sha_pinning_required`) and the repository's default workflow token
permissions/PR-approval restriction. Both had been toggled directly through
the GitHub web UI, so their state existed only on GitHub — undocumented and
unrepresented in any committed configuration.

A new local Terraform root, `/home/julian/projects/github-infra`, was started
to bring GitHub-side repository configuration under Terraform for `dyndns`
and a scratch repository `testing`. A read-only audit against the live
GitHub account (`gh api`, all 21 repositories) confirmed the scope of what
was still missing: the two settings above, plus a third repository,
`whoami`, that has its own pre-existing manual protection (a disabled
ruleset, non-default merge settings, unrestricted Actions permissions) with
no Terraform representation at all.

## Decision

Use `github-infra` as the Terraform root for GitHub repository configuration,
separate from `dyndns` (workload) and `aws-account-bootstrap` (GitHub OIDC
provider and AWS IAM roles). Each managed repository is declared once in
`github-infra/config.yml` and provisioned through a shared `./modules/repo`
module covering: repository settings, a default-branch ruleset, Actions
variables, a `production` environment, the Actions allowed-actions allowlist
(including `sha_pinning_required`), and repository default workflow
permissions (`github_workflow_repository_permissions`).

The two previously-manual settings were closed by importing the
already-live GitHub state rather than recreating it: `sha_pinning_required`
was first added as an explicit per-repository value on the existing
`github_actions_repository_permissions` resource (`true` for `dyndns`,
`false` for `testing`, matching their live values with zero diff), and the
previously entirely untracked `github_workflow_repository_permissions`
resource was absorbed with a `terraform import` block per repository, then
the satisfied import blocks were deleted. `terraform plan` showed only the
two imports (0 creates, 0 changes, 0 destroys) before applying, and a
post-apply `gh api` spot-check confirmed the tracked and live values still
match.

`sha_pinning_required` was then simplified from a per-repository value to a
hardcoded `true` for every repository the module manages, matching the
baseline already applied manually to `dyndns` by
`dyndns/scripts/protect-repository.sh`. `testing`'s prior `false` value was a
scratch-repo artifact rather than a deliberate setting, so it was allowed to
converge to `true` as well; `terraform plan` showed exactly that one
in-place update before it was applied.

`whoami` is a legacy personal repository the user may archive or delete; it
is **intentionally excluded** from `github-infra` and left fully unmanaged
until that decision is made.

`github-infra` currently uses Terraform's default local state backend
(not the shared encrypted S3 backend used by every other Terraform root in
this workspace). This is a known, deliberately deferred gap, not an
oversight — migrate it before treating this root as production-critical.

## Consequences

- GitHub-side repository protection for `dyndns` and `testing` is now fully
  declared in Terraform; no repository setting for those two repos exists
  only as an undocumented manual GitHub UI change.
- Adding a new repository (with or without pre-existing manual
  configuration) follows a documented import-then-clean-up workflow, see
  `github-infra/README.md`.
- `whoami`'s manual protection remains completely outside version control
  and outside this workspace's inventory boundary discussion until an
  explicit decision is made about that repository's future.
- `github-infra`'s state is not yet protected by the shared S3 backend, so
  it currently carries the same loss/drift risk as any local-only Terraform
  state; this is tracked as backlog work, not resolved by this decision.
