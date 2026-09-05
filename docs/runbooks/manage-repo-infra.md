# Runbook: Manage Repository Infrastructure

Status: current. Owning repository:
`/home/julian/projects/github/repo-infra` (local Git `main`, no GitHub remote,
shared encrypted S3 backend, key `repo-infra/terraform.tfstate`).

## Scope

`repo-infra` (renamed from `github-infra`; absorbed the formerly separate
`aws-account-bootstrap` root on 2026-08-21, see
[ADR 0010](../adr/0010-consolidate-repo-and-identity-control-plane.md))
declares GitHub-side repository configuration in `config.yml` and, for
repositories that need AWS deploy credentials, their AWS IAM permissions in
`aws_policies.tf`. Both are provisioned through a shared `./modules/repo`
module:

- GitHub side, always: repository settings, a default-branch ruleset, Actions
  variables, a `production` environment, the Actions allowed-actions
  allowlist plus `sha_pinning_required`, and repository default workflow
  permissions (`github_workflow_repository_permissions`).
- AWS side, only for repositories with an `aws_policies.tf` entry: the
  account-wide GitHub Actions OIDC provider (a single root-level resource),
  per-repository IAM plan/apply roles and policies, and the
  `AWS_ACCOUNT_ID`/`AWS_ROLE_ARN`/`AWS_PLAN_ROLE_ARN` Actions variables set
  from the real created role ARNs.

Currently managed: `dyndns` (GitHub settings + AWS deploy roles), `testing`
(GitHub settings only). `whoami` has pre-existing manual protection but is
intentionally excluded pending a separate archive/delete decision — do not
add it without that decision being made first.

## Standard validation

```bash
cd /home/julian/projects/github/repo-infra
terraform fmt -check -recursive
terraform validate
terraform plan
```

`terraform plan`/`apply` require both a `GITHUB_TOKEN` and a trusted,
short-lived AWS identity capable of managing IAM roles/policies and the OIDC
provider — this root must run only from a trusted local controller, never
through GitHub Actions or a role it manages. Review the plan output before
applying. Any create, replace, or destroy that was not explicitly intended is
a stop condition.

## Adding a new repository that deploys to AWS

1. Add an entry to `config.yml` (GitHub-side settings).
2. Add a matching entry to `aws_policies.tf`'s `local.aws_policies` map with
   that repository's `state_key` and the IAM statements its deployment needs.
   AWS permissions are inherently deployment-specific and can't be derived
   automatically — this is the one thing you still write yourself. Everything
   else (OIDC trust, state-bucket access, role creation, wiring the resulting
   ARNs into the repo's Actions variables) is automatic.
3. `terraform plan`, review, `terraform apply`.

A repository with no `aws_policies.tf` entry gets no AWS role at all.

## Adding a repository with pre-existing manual configuration

This is the pattern used to absorb `dyndns`'s original manual GitHub
settings, later its manually-set `sha_pinning_required` and default workflow
permissions (see
[ADR 0009](../adr/0009-terraform-managed-github-repository-protection.md)),
and — for the AWS side — the 5 IAM/OIDC resources absorbed from
`aws-account-bootstrap` (ADR 0010).

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
   For AWS resources it already has, also inspect them read-only (e.g.
   `terraform state show` on the source root before migrating, or
   `aws iam get-role`/`get-role-policy` for hand-created roles).
2. Add the repository to `config.yml` (and `aws_policies.tf` if it has AWS
   access) with values matching its live configuration exactly — a zero-diff
   import target avoids accidentally changing production settings on first
   apply. Trust policies (`assume_role_policy`) must match byte-for-byte;
   cosmetic fields like an IAM role `description` are lower-risk and can
   differ if reviewed.
3. Add temporary `import { to = ..., id = ... }` blocks in `main.tf` for each
   resource that already exists live but is not yet in Terraform state. IAM
   role import IDs are the role name; inline role policy import IDs are
   `<role-name>:<policy-name>`; the OIDC provider's import ID is its ARN.
4. Run `terraform plan` and confirm it shows only imports — 0 to add, 0 to
   change, 0 to destroy (or only expected, reviewed cosmetic changes). If the
   plan proposes changing a trust policy or any live resource unexpectedly,
   stop and re-check the live values; do not apply a plan that would alter
   production configuration as a side effect of an import.
5. `terraform apply`.
6. Delete the now-satisfied `import` blocks.
7. Re-run `terraform plan` and confirm "No changes."
8. Spot-check with the same read-only calls from step 1 to confirm the
   tracked and live values still match.

If the resource is moving from another Terraform root (as with
`aws-account-bootstrap`), also add `removed { destroy = false }` blocks in
that source root and apply there once the destination import is verified,
following the same no-destroy handoff pattern as
[Migrate AWS account bootstrap ownership](migrate-aws-account-bootstrap.md).

## Adding a new repository with no pre-existing configuration

Add it to `config.yml` with no `import` blocks needed; `terraform apply` will
create all resources fresh.

## Rollback

There is no dedicated rollback playbook for this root. To stop managing a
repository, remove its `config.yml` (and `aws_policies.tf`) entries and run
`terraform plan` to confirm the expected destroys, then apply deliberately —
this will remove the managed ruleset, Actions permissions, environment,
variables, and (if configured) IAM roles/policies from the live repository
and AWS account. Review the plan carefully before applying a removal; the
OIDC provider and IAM roles carry `prevent_destroy`, so a plan that proposes
destroying them will fail rather than silently proceeding.
