# ADR 0006: Manage AWS Deployment Identities from a Local-Only Control Plane

- Status: Accepted; migration pending
- Date: 2026-08-19

## Context

The `dyndns` Terraform root currently owns the account-wide GitHub Actions OIDC
provider and the repository-bound plan/apply roles that authenticate its own
workflows. Those roles intentionally cannot modify their own trust or
permissions, so IAM/OIDC changes already require a separate privileged local
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
- The initial transfer requires a guarded cross-state handoff. The source uses
  `removed` blocks with `destroy = false`; the destination imports the same
  live objects. Any proposed create, replacement, or destroy blocks migration.
