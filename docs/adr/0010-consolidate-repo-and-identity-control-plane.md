# ADR 0010: Consolidate GitHub repository and AWS identity control planes

- Status: Accepted
- Date: 2026-08-21
- Deployment status: Applied

## Context

ADR 0006 created `aws-account-bootstrap` as a dedicated local-only Terraform
root for the account-wide GitHub Actions OIDC provider and per-repository AWS
IAM plan/apply roles, kept separate from `dyndns`'s workload state. Separately,
`repo-infra` (formerly `github-infra`, see ADR 0009) manages GitHub-side
repository settings (branch rulesets, Actions permissions, environments).

Adding a new GitHub repository that deploys to AWS required editing both
roots: `aws-account-bootstrap` to create its IAM roles, `repo-infra` to create
the GitHub repository and set its Actions variables. `aws-account-bootstrap`
was also **100% hardcoded for `dyndns`** — a single file, no parameterization,
no documented procedure for a second repository. The two roots were coupled
only by convention, not a real dependency: `aws-account-bootstrap`'s README
told the operator to copy its output ARNs into the GitHub repo Variables UI by
hand, and `repo-infra` independently guessed the same ARNs by string
convention plus a hardcoded AWS account ID. In practice there were three
independent copies of the same contract (the hardcoded IAM config, the
guessed ARN strings, and whatever was actually pasted into GitHub), with no
way for any of them to detect drift from the others.

## Decision

Merge `aws-account-bootstrap`'s resources into `repo-infra`, in a single
Terraform state. `modules/repo` gained AWS sub-resources gated on an optional
per-repository `aws` input: IAM plan/apply roles with a trust policy built
from the repository's own computed `repo_id` (no more hardcoded numeric repo
IDs), a generic permissions baseline (state-bucket access, self-role/OIDC
read), plus that repository's specific IAM statements supplied in
`aws_policies.tf`, and `github_actions_variable` resources set from the real
created role ARNs.

This keeps ADR 0006's core concern fully intact: the AWS identity control
plane is still a distinct Terraform state from `dyndns`'s workload state, so
ordinary DynDNS plans still can't touch administrative IAM resources, and a
compromised DynDNS workflow still can't modify its own trust or permissions.
What changes is only *which* control-plane repository the identity resources
live in, and that AWS permissions — which are inherently deployment-specific
and can't be derived automatically — are now the one thing a human still
writes per new AWS-deploying repository, in one file, in one repository,
instead of duplicating a whole Terraform root.

`aws-account-bootstrap` is retired. It was local-only with no remote and only
4 commits; nothing of value is lost by deleting it, since the full migration
record lives in `repo-infra`'s commit history and the dated handover in
`home-infra-ai-context`. **ADR 0006 is superseded by this decision** — its
consequences remain substantively accurate, but its repository name/location
is no longer current.

## Migration

Modeled on ADR 0006's own no-destroy handoff pattern: `repo-infra` absorbed
the 5 live AWS resources (the OIDC provider, both IAM roles, both inline role
policies) via `import` blocks, verified with a `terraform plan` showing only
those 5 imports plus 2 purely cosmetic IAM role `description` changes (trust
policies were byte-identical — confirmed before applying) and 0 unexpected
create/change/destroy. `aws-account-bootstrap` then relinquished the same 5
resources with `removed { destroy = false }` blocks. After the handoff, a
`gh api` cross-check confirmed `dyndns`'s live GitHub Actions variables
(`AWS_ACCOUNT_ID`, `AWS_ROLE_ARN`, `AWS_PLAN_ROLE_ARN`) still match the real
role ARNs exactly — its CI required no changes. `repo-infra`'s state was then
migrated from a local backend to the shared encrypted S3 bucket (a
pre-existing, separately tracked gap — a local backend was an acceptable risk
for GitHub-only settings but not for holding the AWS IAM/OIDC control plane).

## Consequences

- Adding a new AWS-deploying GitHub repository is one `config.yml` entry plus
  one `aws_policies.tf` entry, then one `terraform plan`/`apply` in one
  repository — not edits across two separate Terraform roots.
- The guessed-ARN-string and hardcoded-account-ID logic in `repo-infra` is
  gone; `AWS_ROLE_ARN`/`AWS_PLAN_ROLE_ARN`/`AWS_ACCOUNT_ID` are now real
  Terraform references to the roles this same apply creates, eliminating the
  drift risk between three independent copies of the same contract.
- `repo-infra` now carries the same trust requirements ADR 0006 established:
  run only from a trusted local controller with a human-selected, short-lived
  AWS identity, never through GitHub Actions or a role it manages.
  `prevent_destroy` remains set on the OIDC provider and both IAM roles.
- `whoami`'s pre-existing manual GitHub protection remains explicitly out of
  scope (see ADR 0009), unaffected by this change.
