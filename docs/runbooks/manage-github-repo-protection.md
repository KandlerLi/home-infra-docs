# Runbook: Manage GitHub Repository Protection

Status: current. Owning repository:
`/home/julian/projects/github-infra` (local Git `main`, no GitHub remote,
local-only Terraform state).

## Scope

`github-infra` declares GitHub-side repository configuration in
`config.yml` and provisions it through a shared `./modules/repo` module:
repository settings, a default-branch ruleset, Actions variables, a
`production` environment, the Actions allowed-actions allowlist plus
`sha_pinning_required`, and repository default workflow permissions
(`github_workflow_repository_permissions`).

Currently managed: `dyndns`, `testing`. `whoami` has pre-existing manual
protection but is intentionally excluded pending a separate archive/delete
decision — do not add it without that decision being made first. This root
does not manage the account-wide GitHub OIDC provider or AWS IAM roles; those
belong to `/home/julian/projects/aws-account-bootstrap`.

## Standard validation

```bash
cd /home/julian/projects/github-infra
terraform fmt -check -recursive
terraform validate
terraform plan
```

Review the plan output before applying. Any create, replace, or destroy that
was not explicitly intended is a stop condition.

## Adding a repository with pre-existing manual configuration

This is the pattern used to absorb `dyndns`'s original manual settings and,
later, its manually-set `sha_pinning_required` and default workflow
permissions (see
[ADR 0009](../adr/0009-terraform-managed-github-repository-protection.md)).

1. Read the repository's current live settings with read-only `gh api` calls
   before writing any Terraform:
   ```bash
   gh api repos/{owner}/{repo}
   gh api repos/{owner}/{repo}/branches/{branch}/protection
   gh api repos/{owner}/{repo}/rulesets
   gh api repos/{owner}/{repo}/actions/permissions
   gh api repos/{owner}/{repo}/actions/permissions/workflow
   gh api repos/{owner}/{repo}/environments
   ```
2. Add the repository to `config.yml` with values matching its live
   configuration exactly (a zero-diff import target avoids accidentally
   changing production GitHub settings on first apply).
3. Add temporary `import { to = ..., id = ... }` blocks in `main.tf` for each
   resource that already exists live but is not yet in Terraform state.
4. Run `terraform plan` and confirm it shows only imports — 0 to add, 0 to
   change, 0 to destroy. If the plan proposes any change to the live
   resource, stop and re-check the live values; do not apply a plan that
   would alter production GitHub configuration as a side effect of an
   import.
5. `terraform apply`.
6. Delete the now-satisfied `import` blocks (leaving a satisfied import block
   in place is inert but confusing; clean it up immediately after applying).
7. Re-run `terraform plan` and confirm "No changes."
8. Spot-check with the same `gh api` calls from step 1 to confirm the tracked
   and live values still match.

## Adding a new repository with no pre-existing configuration

Add it to `config.yml` with no `import` blocks needed; `terraform apply` will
create all resources fresh.

## Rollback

There is no dedicated rollback playbook for this root. To stop managing a
repository, remove its entry from `config.yml` and run `terraform plan` to
confirm the expected destroys, then apply deliberately — this will remove
the managed ruleset, Actions permissions, environment, and variables from the
live repository. Review the plan carefully before applying a removal.
