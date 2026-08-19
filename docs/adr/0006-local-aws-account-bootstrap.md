# ADR 0006: Manage AWS Deployment Identities from a Local-Only Control Plane

- Status: Accepted and deployed
- Date: 2026-08-19
- Deployment date: 2026-08-19

## Context

The `dyndns` Terraform root originally owned the account-wide GitHub Actions
OIDC provider and the repository-bound plan/apply roles that authenticate its
own workflows. Those roles intentionally could not modify their own trust or
permissions, so IAM/OIDC changes already required a separate privileged local
apply.

Keeping the resources in the workload state makes ownership less explicit and
causes ordinary DynDNS plans to include administrative control-plane resources
that CI cannot change.

## Decision

Create `/home/julian/projects/aws-account-bootstrap` as a private, local-only
Terraform root. It owns the account-wide GitHub OIDC provider and both DynDNS
automation roles with their inline policies. It uses the isolated S3 backend
key `aws-account-bootstrap/terraform.tfstate`.

Never run this root in GitHub Actions or through a role that it manages. Use a
human-selected, short-lived local administrative or narrowly scoped bootstrap
identity.

Keep `/home/julian/projects/terraform-state` focused on the central S3 backend
bucket. Keep application resources and the Lambda runtime role in `dyndns`.

## Consequences

- IAM trust and deployment permissions have an explicit administrative owner.
- A compromised DynDNS workflow cannot change its own plan/apply identity.
- Permission changes require a reviewed local plan and apply.
- The control-plane state becomes a high-trust state key and must remain
  inaccessible to workload roles.
- Changes to workload names or permissions may require coordinated changes in
  both repositories.
- The initial transfer required a guarded cross-state handoff. The source used
  `removed` blocks with `destroy = false`; the destination imported the same
  live objects. Any proposed create, replacement, or destroy was a stop
  condition.

## Deployment outcome

DynDNS PR #8 merged as `d8d6541`. Its checks and protected apply succeeded,
the existing objects were imported locally, and final no-change plans and role
tests were confirmed. The workload and identity control planes now use
separate state keys.
