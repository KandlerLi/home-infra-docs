# ADR 0017: Monitoring split by cost — homeserver-hosted, one AWS-side reachability check

- Status: Accepted
- Date: 2026-08-23
- Deployment status: `aws/homeserver-health-check` built and locally
  validated, not yet applied. The homeserver-hosted half is a separate,
  larger follow-up, not yet started.

## Context

`home-infra`'s known gaps have long included "no monitoring role or
monitoring containers." Picking that up, the user asked to keep AWS cost
down: rather than running a monitoring stack (metrics, dashboards,
alerting) in AWS, most of it should run on the homeserver itself, where
compute is already paid for and free to use. But monitoring that lives
entirely on the homeserver has a structural blind spot: it cannot tell
anyone the homeserver itself is unreachable, since the thing doing the
alerting is the thing that would be down.

## Decision

**Split by where the check has to run, not by what it checks.** AWS hosts
exactly one thing: an external reachability check for the home network,
implemented as `aws/homeserver-health-check` — a Route53 health check
against the bare `jkandler.de` apex (not `www.jkandler.de`, which is
CloudFront/S3 and would only prove AWS itself is reachable) every 30
seconds, a CloudWatch alarm on that health check's status, and an SNS
email subscription for both the unhealthy transition and the recovery.
Everything else that monitoring would normally cover — host health,
container health, individual service reachability, certificate/DNS
expiry — is planned to run as a Prometheus + Grafana stack on the
homeserver itself, for free, as a separate follow-up.

This follows the same repo/Terraform/CI shape every other AWS-deploying
repo in this workspace uses: its own `infra/` Terraform root, GitHub
repository and OIDC roles provisioned by `github/repo-infra`, IAM
policies scoped tightly to the three resource types actually touched
(Route53 health checks, one named CloudWatch alarm, one named SNS topic),
and a self-hosted-runner CI/CD pipeline gated behind the `production`
environment's reviewer approval before any real `terraform apply`.

## Consequences

- Ongoing AWS cost for monitoring stays at roughly $0.50-1/month (one
  Route53 health check plus negligible CloudWatch/SNS usage) regardless of
  how large the homeserver-side stack eventually grows.
- If the home network goes down, the AWS-side check is the only thing
  that can still notify the user — by design, since it is the only piece
  of monitoring that doesn't depend on the homeserver being up.
- If AWS itself has an outage, or the SNS email never gets confirmed
  (a manual, unavoidable step), homeserver reachability alerting is
  silently unavailable until that's fixed; there's no redundancy on the
  AWS side, since redundancy wasn't the goal — cost was.
- Every other future monitoring capability (dashboards, per-service
  alerts, cert expiry, etc.) is scoped to the homeserver-side follow-up,
  not to this repo — `aws/homeserver-health-check` is deliberately kept
  this small and is not expected to grow into a general-purpose AWS
  monitoring stack.
