# ADR 0018: A scoped IAM identity replaces root for local Terraform applies

- Status: Accepted
- Date: 2026-08-23
- Deployment status: Built and locally validated in
  `bootstrap/terraform-state/operator.tf` (see the 2026-08-23 Update
  below), not yet applied

## Context

`terraform-state` and `repo-infra` are both control-plane roots that are
deliberately never applied through GitHub Actions or any role they
themselves manage (ADR 0006/0010) — they run only from a trusted local
controller. In practice, "trusted local controller" has meant the AWS
account root user throughout this workspace's history. Root has MFA
enabled and no long-lived access keys (the account's other no-MFA root-
adjacent risk, an active key on the IAM user `julian`, was already found
and remediated by the 2026-08-23 AWS account audit — see
`home-infra-ai-context/handovers/2026-08-23-aws-account-audit.md`), so
this was never an active credential leak. But it's still the single most
powerful identity in the account, used routinely for what is, in scope,
a narrow, well-defined set of operations.

## Decision

**Reuse the existing IAM user `julian`** (created 2021, previously
managed only by hand, carrying `AdministratorAccess`) as the day-to-day
local-Terraform identity, rather than creating a new dedicated user.
`AdministratorAccess` is replaced with a policy scoped to exactly what
running `terraform-state` and `repo-infra` locally does today: the shared
Terraform state bucket, the GitHub Actions OIDC provider, and IAM role/
policy management restricted to the `*-github-actions`/`*-github-plan`
naming convention `repo-infra`'s `modules/repo` uses for every
repository's own CI roles — never a bare `iam:*` grant.

**Tightly scoped to today's exact usage, not broad-but-safe.** The
alternative considered was AWS-managed `PowerUserAccess` (blocks IAM/
Organizations by default) plus the same narrow IAM carve-out — simpler to
maintain, since it wouldn't need widening every time a new AWS service
shows up somewhere in the workspace. Chosen instead: hand-enumerate only
what's actually used. This is more maintenance (the next time
`terraform-state` or `repo-infra` themselves start touching a new AWS
action, this policy needs a matching update — as already happened once
during `homeserver-health-check`'s IAM policy, where a live apply
surfaced a missing `sns:GetSubscriptionAttributes` grant), but it means
`julian`'s day-to-day permissions never exceed a reviewable, explicit
list, and a future compromise of this identity is bounded by exactly what
it can be seen to do, not by what `PowerUserAccess` happens to allow.

**Kept out of `repo-infra`, in a root of its own — originally a new
standalone `bootstrap/operator-identity`, later merged into
`terraform-state`; see the 2026-08-23 Update below.** `julian`'s policy
deliberately grants no IAM permissions over IAM users, groups, or itself
— the entire point is that this identity can never widen its own access.
That guarantee only holds if the resources managing `julian`'s own
user/policy live somewhere `julian` has zero access to. If they lived
inside `repo-infra`, every plan there — even for something unrelated,
like adding a new repository to `config.yml` — would try to refresh the
identity resources too and fail under `julian`'s own restricted
credentials, since `julian` can't read its own user/policy resources by
design. A separate root with its own state file (which `julian`'s policy
also doesn't grant access to) avoids that: this identity can only ever be
planned or applied as root.

**`AdministratorAccess` removal is a deliberate second apply, not a
one-shot switch.** Terraform can only stop managing a resource already in
its state, so the policy attachment is first imported (recording it, no
live change) in the same apply that attaches the new scoped policy,
verified to work, and only then removed from configuration in a follow-up
apply that actually detaches it. See the root's own README for the exact
two-step sequence.

**Console login and MFA enrollment stay manual, on purpose.** A login
profile password and an MFA device both need direct human interaction —
neither should flow through an agent session or get written anywhere,
committed or not.

## Consequences

- Once applied, day-to-day `terraform-state`/`repo-infra` work uses
  `julian` (via `aws login`'s IAM-user sign-in) instead of root.
- `julian` deliberately **cannot** do the kind of ad-hoc, cross-service
  read-only AWS CLI verification (budgets, Route 53 health checks,
  CloudWatch alarms, SNS, IAM audits) that root has been used for
  throughout this workspace's history to confirm live deployments worked.
  That still needs root, or a separate widening of this policy, done
  deliberately — not a fallback to root out of convenience.
- Every future repository's own AWS permissions (its `aws_policies.tf`
  entry) continue to apply only to *that repository's* GitHub Actions
  roles, entirely independent of this identity — adding a new repo never
  requires touching this identity's own management.
- Root remains necessary for exactly one thing going forward: applying
  the root that manages `julian`'s own identity, whenever `julian`'s own
  access needs to change (see the Update below for where that now lives).

## Update (2026-08-23): merged into `terraform-state`, not a standalone root

Built initially as its own root, `bootstrap/operator-identity`. Before
being applied, the user pointed out that it and `terraform-state` share
the same operational profile: both are foundational, applied only by
root, and touched essentially once at bootstrap — unlike `repo-infra`,
which `julian` applies routinely. A third such root, on top of
`terraform-state` and `repo-infra`, was unnecessary overhead for
something at the same trust level and cadence as one that already exists.

`operator.tf`'s resources moved into `bootstrap/terraform-state`
unchanged in substance, with one tightening: `julian`'s policy no longer
grants access to `terraform-state/*` state or to bucket management,
since `julian` never applies `terraform-state` itself — only `repo-infra`.
The core guarantee this ADR's Decision section describes — that `julian`
has zero access to the resources managing its own identity — holds
exactly the same way, just inside `terraform-state` instead of a
dedicated root. The short-lived `operator-identity` GitHub repository
and local directory were both removed the same day.
