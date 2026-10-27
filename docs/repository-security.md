# Secure Repository and Deployment Blueprint

Source example: [`dyndns`](https://github.com/KandlerLi/dyndns) at `df2fa76` on 2026-08-19.

This records how `KandlerLi/dyndns` was secured and how to adapt the pattern to
other repositories. It is a blueprint, not a script to run unchanged. Resource
names, repository IDs, permissions, status contexts, environments, and human
review rules must be reviewed for every new repository.

## Security objectives

The design aims to ensure that:

- changes reach `main` only through an explicitly approved, validated PR
- a new commit invalidates the previous approval
- untrusted fork code never receives cloud credentials
- PR plans can observe but cannot mutate AWS or Terraform state
- production deployment receives short-lived credentials only after a second
  environment approval
- deployment access is narrowed to the intended state key, hosted zone, named
  resources, and necessary service scopes
- GitHub Actions dependencies cannot silently move to different code
- a compromised workflow cannot modify its own AWS trust or deployment role
- application credentials never enter Git, GitHub secrets, Terraform state, or
  access logs

## Layer 1: Repository governance

Since 2026-08-21 most of this is Terraform-managed by
[`repo-infra`](https://github.com/KandlerLi/repo-infra) (`github_branch_protection`,
`github_actions_repository_permissions`, `github_workflow_repository_permissions`,
`github_repository_environment`), not by a script — see that root's README
for the resource-level detail.

`pull_request_creation_policy: collaborators_only` (who may open a PR at
all) and, on public repositories, the fork-PR workflow approval policy
`all_external_contributors` have no `terraform-provider-github` resource
yet (upstream issue #3198). `repo-infra`'s `pr_policy.tf` (added
2026-09-25; first apply still pending when this was written) manages
both for every `config.yml` entry: a
`data "external"` reads the live value with `gh api` on every plan, and a
`terraform_data` + `local-exec` sets it only when it differs, so drift
shows up in `plan` (see `decisions.md`, "local-exec is allowed only with a
live read as its trigger"). Before that, the setting was a manual step
here and had been missed on 13 of 14 repositories.

One thing remains outside Terraform, because there is currently no IaC
path for it:

- Removing every direct collaborator except the owner — no Terraform
  resource in this workspace declares an authoritative empty collaborator
  set today.

This and `pull_request_creation_policy` used to live in a repository-specific script
(`dyndns/scripts/protect-repository.sh`), but it was deleted on
2026-09-02: even narrowed to just these two settings, it was still an
imperative script wrapping `gh api` against live state, the shape the
standing IaC-only rule (`decisions.md`)
forbids regardless of how small the remaining scope is. For a new
repository following this blueprint, prune collaborators by hand —
authenticate as the intended owner, confirm admin permission on the
repository, `gh api --method DELETE
repos/<owner>/<repo>/collaborators/<login>` for each non-owner direct
collaborator — as a deliberate one-off at setup time, not as a script kept
around to re-run.

The bullet lists below describe the security design itself, which hasn't
changed — only which tool enforces it has.

### Ownership and collaborators

- `CODEOWNERS` assigns all paths to `@KandlerLi`.
- Only the owner should remain as a direct collaborator; prune the rest by
  hand as described above.
- Pull-request creation is limited to collaborators, and fork-PR workflows
  on public repositories always wait for owner approval (`repo-infra`'s
  `pr_policy.tf` — see above).
- Deploy keys, webhooks, and installed GitHub Apps are listed for manual review.

The collaborator removal is intentionally destructive. A reusable version must
accept and display an explicit allowlist, support an audit/dry-run mode, and
require confirmation before removing access.

### Merge and branch policy

Enforced by `repo-infra`'s `github_branch_protection` resource, applied as a
hardcoded baseline to every repository it manages. For `main`:

- pull requests are required
- status checks must be current with the branch (`strict: true`)
- required contexts are `Validate` and `Terraform plan`
- branch protection applies to administrators
- conversations must be resolved
- signed commits are required
- linear history is required
- only squash merging is enabled
- force pushes and branch deletion are disabled
- merged branches are deleted automatically
- automatic merging is disabled

GitHub does not allow a sole author to approve their own PR review, so
native `required_pull_request_reviews` approval was never viable here — the
repository's `pull_request_creation_policy: collaborators_only` plus a
single collaborator (the owner) means every PR is self-authored. An earlier
design used a separate exact-commit status-check workflow
(`owner-approval.yml`, requiring an explicit `/approve` comment before
merge) to work around that. It was removed on 2026-08-21: since no one
other than the owner can merge regardless (merging requires GitHub write
access, and the owner is the only collaborator), the extra manual gate
protected only against the owner's own mistakes, not against anyone else —
and the separate `production` environment reviewer gate (approval required
before the apply job receives AWS credentials, see Layer 3) already provided
that checkpoint after merge, closer to the point of actual risk.

The same reasoning was applied a second time on 2026-09-07, to that
`production` environment reviewer gate itself: an AI agent may commit and
open PRs on this repository, but merging remains a deliberate human action
the agent never performs (the owner's standing rule) — so the
separate "approve deployment" click was reviewing nothing a second pair of
eyes hadn't already seen at merge time, for the same single-owner reason the
2026-08-21 removal already established. The gate was removed; the
environment resource itself stayed, since it's also the AWS OIDC trust
boundary (see Layer 3), unrelated to the reviewer gate. The current flow is:
open a PR, review the `Terraform plan` job's speculative plan output in the
checks tab, merge once `Validate` and `Terraform plan` are green — Terraform
then applies immediately, with no separate approval step.

`CODEOWNERS` still names `@KandlerLi` as sole code owner for documentation
purposes, but `require_code_owner_reviews` is `false` — it isn't enforced.

## Layer 2: GitHub Actions supply-chain controls

Repository Actions settings enforce:

- default workflow token permissions are read-only
- workflows cannot approve pull-request reviews
- only a small allowlist of action families is permitted
- SHA pinning is mandatory

The current allowlist is:

```text
actions/checkout@*
aws-actions/configure-aws-credentials@*
hashicorp/setup-terraform@*
```

Workflow files still pin each action to a full commit SHA and include the human
version as a comment. For another repository, allow only the action families it
actually needs and pin every use to a reviewed full SHA.

Jobs also have explicit timeouts and least-privilege `permissions` blocks.

## Layer 3: Separate validation, plan, and apply

### Credential-free validation

Every PR runs:

- `terraform fmt -check -recursive`
- `terraform init -backend=false -input=false`
- `terraform validate`
- Python standard-library unit tests

This job has only `contents: read` and no OIDC token. Basic validation therefore
works for forks without exposing cloud access.

### Read-only speculative plan

The plan job runs only when the pull request's head repository is the same as
the base repository. Fork PRs are skipped.

That `if:` condition is a convenience, not a security boundary: for
`pull_request` events GitHub runs the workflow file from the PR itself, so
a fork PR can delete the condition. What actually keeps untrusted code off
the self-hosted runners is that only collaborators can open PRs and that
fork-PR workflows need owner approval (both enforced by `pr_policy.tf`).

It receives `contents: read` and `id-token: write`, assumes a dedicated
read-only AWS role, initializes the real backend, and runs:

```bash
terraform plan -input=false -lock=false -no-color
```

The role may read the one Terraform state object and metadata for the specific
managed resources. It cannot write/delete state or locks, read the secret value,
change Route 53, deploy Lambda/API Gateway, or modify IAM.

### Protected production apply

The apply workflow runs on `main` pushes or manual dispatch and targets the
GitHub `production` environment. That environment:

- permits only protected branches
- is referenced by the AWS OIDC trust subject

It no longer requires a reviewer (removed 2026-09-07 — see Layer 1): the
environment resource is kept solely as the AWS OIDC trust boundary
(`token.actions.githubusercontent.com:sub` must equal
`repo:.../environment:production` for the apply job to assume its role), not
as an approval gate. The job receives its production OIDC identity
immediately once it starts running on `main`, with no manual click in
between. For a team repository, a reviewer requirement (with
`prevent_self_review: true` and an independent reviewer) would normally be
worth restoring.

The workflow:

- verifies required non-secret variables before authentication
- requests only `contents: read` and `id-token: write`
- checks the allowed AWS account ID while assuming the role
- creates a saved Terraform plan
- applies that exact saved plan
- uses non-interactive Terraform settings
- serializes all Terraform runs with concurrency and never cancels an in-flight
  apply

The repository variables are account ID, apply-role ARN, plan-role ARN, and
Route 53 zone ID. They are identifiers, not long-lived AWS credentials.

## Layer 4: Short-lived, immutable OIDC trust

There are no AWS access keys in GitHub. GitHub exchanges an OIDC token for a
short-lived AWS session.

Separate roles exist for:

- pull-request planning
- production deployment

Trust conditions require the `sts.amazonaws.com` audience and an exact subject.
The subject is bound to the repository owner name and immutable numeric owner
ID, repository name and immutable numeric repository ID, plus either:

- `pull_request` for the plan role
- `environment:production` for the apply role

The exact subject format depends on the GitHub OIDC subject customization in
use. For a new repository, query its numeric IDs, inspect/verify the token claim
format, and generate the IAM condition deliberately. Do not copy the `dyndns`
IDs or assume a name-only subject is equivalent.

The account-wide GitHub OIDC provider has `prevent_destroy = true`. Its
Terraform ownership, together with both repository roles and policies, was
moved without destruction to the local-only `aws-account-bootstrap` root on
2026-08-19, then to [`repo-infra`](https://github.com/KandlerLi/repo-infra) on 2026-08-21 when
that root was retired and its resources consolidated with GitHub repository
settings management (see ADR 0010 in `home-infra-docs`).

## Layer 5: Least-privilege AWS and state access

Terraform state uses an encrypted S3 backend with native S3 lock files. IAM is
scoped to the repository's exact state and lock object keys.

The shared bucket is currently defined by the
[`terraform-state`](https://github.com/KandlerLi/terraform-state) repository, applied only locally.
Its ignored local state backup and backend metadata must be treated as
sensitive; do not inspect, copy, publish, delete, or migrate them as part of
routine documentation work. Before publishing it or using this pattern for
further repositories, apply the same review and CI protections described here,
using a recovery-aware plan that preserves the existing remote and local state.

The apply role is limited by service, region, account, names, hosted-zone ID,
and state prefix where AWS supports those dimensions. Notably:

- Route 53 access is limited to the selected hosted zone
- Lambda access is limited to the configured function name
- logs are limited to the two managed log groups, except the unavoidable
  describe operation
- Secrets Manager access is limited to the configured secret-name prefix
- IAM mutation is limited to the Lambda runtime role
- the automation roles and GitHub OIDC provider are read-only to the deployment
  role

Keeping the automation roles read-only prevents the CI deployment role from
expanding its own permissions or changing its trust. Ownership places
the provider and both roles/policies in [`repo-infra`](https://github.com/KandlerLi/repo-infra)
(alongside GitHub repository settings management). That root has no GitHub
Actions workflow and requires a human-selected, short-lived local bootstrap
identity plus a GitHub token. This creates an explicit
bootstrap/control-plane boundary while `dyndns` continues to own its workload
resources and Lambda runtime role.

The plan role has only the read calls Terraform needs. It can inspect secret
metadata but cannot call `GetSecretValue`.

### Residual limitations to understand

- API Gateway management uses a region-wide API Gateway ARN because of the
  service/API actions Terraform needs; this is broader than the named Lambda,
  secret, hosted-zone, and state scopes.
- Both the PR `/approve` status and production environment approval are made by
  the same sole owner. This provides deliberate two-stage friction but not
  independent separation of duties.
- Same-repository PR branches can obtain the read-only plan role. Restricting
  collaborators to the owner currently limits who can create such branches.
- The GitHub OIDC provider is account-wide even though role trust is
  repository-specific.
- Required status contexts, GitHub Apps, hooks, and administrator account
  security remain part of the trust boundary.

For a team or higher-risk repository, require independent reviewers, prefer a
GitHub ruleset with bypass restrictions, and narrow any service wildcard that
the target platform allows.

## Layer 6: Application/runtime controls

Repository security is not the only protection. The deployed DynDNS service
also constrains the operation itself:

- API Gateway exposes only `GET /nic/update`.
- The default rate is one request per second with burst two.
- Access logs include request ID, route, status, latency, and integration error,
  but exclude authorization and query strings.
- HTTP Basic credentials live in AWS Secrets Manager.
- Terraform creates the secret container but not its value, keeping credentials
  out of configuration and Terraform state.
- Lambda compares credentials with `hmac.compare_digest`.
- Lambda accepts only the configured apex hostname.
- It accepts only a syntactically valid, globally routable IPv4 address.
- The runtime role can update records only in the selected hosted zone and read
  only its own secret.
- The hosted-zone ID is checked against the configured domain.
- CloudWatch retention is bounded.
- The Secrets Manager resource has a seven-day recovery window.
- Unit tests cover valid updates, credential caching, and rejection of wrong
  hostnames, credentials, and private or malformed addresses.

The Lambda currently logs the updated hostname and public IPv4 address. That is
acceptable for operational AWS logs but public dynamic IP snapshots must not be
copied into Git or long-lived handover documentation.

## Secure bootstrap order for another repository

Avoid the OIDC/environment chicken-and-egg problem by using this sequence.
For this workspace, most of it is now handled by
[`repo-infra`](https://github.com/KandlerLi/repo-infra)'s `modules/repo` (see
`home-infra-docs/docs/runbooks/manage-repo-infra.md`) rather than by hand:
the module derives the repository's numeric ID and builds the OIDC trust
subject automatically, and creates the plan/apply roles, state-bucket access,
and `production` environment as soon as the repository has an
`aws_policies.tf` entry. What still requires manual work is exactly what
can't be automated — the repository's specific IAM statements.

If the repository needs a self-hosted GitHub Actions runner (a separate
concern from the AWS/OIDC trust below), set `runner: true` on its
`config.yml` entry and follow `repo-infra/README.md`'s "Adding a repository
with a self-hosted GitHub Actions runner" section — that step stays outside
Terraform, since attaching a runner process is imperative.

1. Add the repository to `repo-infra/config.yml` and, if it needs AWS access,
   an entry in `aws_policies.tf` with its intended cloud account, region,
   resources, state key, and least-privilege IAM statements for its plan and
   apply roles.
2. With a trusted, short-lived local AWS identity plus a GitHub token, run
   `terraform plan`/`apply` in `repo-infra` — it creates/updates the OIDC
   provider (if not already present), the repository's roles, policies, and
   trust-subject construction in one step. Do not run the control plane
   through a managed role. If transferring existing resources, use reviewed
   `destroy = false` removals in the source followed by `import` blocks in
   `repo-infra`, and stop on any create, replacement, or destroy.
3. Keep `terraform-state` limited to the central S3 bucket; `repo-infra`
   uses that same shared backend under its own state key.
4. The GitHub `production` environment and reviewer/branch policy are created
   by the same `terraform apply` above.
5. Add only non-secret repository variables for account/resource/role IDs.
6. Add pinned workflows with explicit permissions and fork-safe conditions.
7. Add exact-commit approval and CODEOWNERS.
8. Audit deploy keys, hooks, Apps, collaborators, Actions settings, and existing
   branch protection before enforcing changes.
9. Apply repository protections, preferably from a parameterized audit-first
   script.
10. Open a harmless test PR. Verify validation, read-only plan, approval reset on
    a new commit, merge protection, production approval, and successful apply.
11. Confirm from cloud audit logs that the plan and apply sessions assumed the
    intended distinct roles and could not exceed their permissions.

Do not bootstrap by merging a workflow that temporarily removes the production
environment gate. Establish the matching AWS trust out of band first, then keep
the environment gate present in the protected branch.

## Adaptation checklist

Replace and verify all of these for a new repository:

- owner login and immutable owner ID
- repository name and immutable repository ID
- default/protected branch
- production environment name and reviewer policy
- required status-context names
- action allowlist and pinned SHAs
- state bucket, state key, lock key, region, and encryption
- plan/apply role names and exact OIDC subjects
- AWS account and allowed-account check
- resource ARNs and every allowed API action
- fork policy and whether plans are safe for same-repository contributors
- collaborator allowlist
- team versus sole-owner approval design
- service-specific tests, limits, logging, secrets, and recovery policy

## Controls that still need periodic manual review

GitHub Apps, deploy keys, webhooks, environment reviewers, repository variables,
OIDC subject customization, cloud audit logs, and account-wide IAM/OIDC
providers can drift outside this repository. Re-run an audit after ownership,
repository visibility, branch, workflow, or cloud-account changes.
