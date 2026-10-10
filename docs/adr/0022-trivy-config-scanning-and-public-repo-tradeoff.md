# ADR 0022: Trivy config scanning across every repo, and the public-repo branch-protection tradeoff

- Status: Accepted
- Date: 2026-09-14/16
- Deployment status: applied and live -- every repo in `github/
  repo-infra`'s own `config.yml` that has a real Terraform CI pipeline
  now requires a passing `trivy-config` check before merge

## Context

Trivy's config scanner (`aquasecurity/trivy-action`, via `gha-common`'s
`trivy-config.yml` reusable workflow) was added to every repo's CI, one
at a time, each repo's own real findings triaged and either fixed or
deliberately suppressed with a documented reason before the check
became a required status check -- never added as a required check
first and left to block merges on unfixed findings.

## Decisions

**Findings fixed per repo, before `trivy-config` became required:**

- `aws-budget`: already clean on its first run (0 findings) -- nothing
  to fix.
- `homeserver-health-check`: needed a dedicated `us-east-1` KMS key for
  its own SNS topic (`aws_kms_key.alerts`, `main.tf`) -- the shared
  workspace KMS key lives in `eu-central-1` and can't be used
  cross-region for an SNS topic that has to live in `us-east-1`.
- `ses-relay`: the SMTP user's own IAM policy moved from being attached
  directly to the IAM user to an `aws_iam_group`/`aws_iam_group_policy`
  pair instead (`main.tf`).
- `secrets-manager`: all 12 instances of AWS-0098 ("Secrets Manager
  should use a customer-managed KMS key") suppressed, not fixed -- see
  that repo's own README.md, "Trivy AWS-0098 (customer-managed key)
  suppressed, not fixed" section, for the full reasoning.
- `k3s-apps`: `k3s-apps#52` fixed DS-0002 (container should not run as
  root) and DS-0026 (missing `HEALTHCHECK`) in
  `containers/sankey-export/Dockerfile` -- a explicit `USER 10010` and
  a `.trivyignore` suppression respectively (a `HEALTHCHECK` has no
  meaning for a `restart_policy: Never` CronJob Pod that's expected to
  exit every run, not stay up as a long-running service).
- `website`: AWS-0132 (bucket should use a CMK) fixed with the shared
  CMK. AWS-0320 (bucket name not DNS-compliant, since `www.jkandler.de`
  has dots) suppressed, not fixed -- deliberately, after a real
  production incident: a rename to a compliant name (`jkandler-website`)
  was attempted live and reverted. Terraform's default
  destroy-then-create replacement order tore down the old bucket's
  policy/access-block/ownership-controls before failing to delete the
  (non-empty) bucket; `force_destroy` didn't help because it wasn't
  recorded on the *old* instance's own state; CloudFront's origin then
  briefly pointed at a bucket that no longer existed; a credential
  expiry mid-apply left a stuck state lock; recovery needed a manual
  `terraform import` plus state surgery. Left suppressed rather than
  retried: the check's own underlying concern (TLS certificate name
  matching on dotted bucket names) never applied here in the first
  place, since this bucket is only ever reached via CloudFront's Origin
  Access Control, never raw virtual-hosted-style HTTPS directly -- not
  worth repeating that incident for a fix with no real security
  benefit.
- `dyndns`, `repo-infra`, `terraform-state`, `k3s-bootstrap`: each
  confirmed clean (0 findings) before the check became required.

**Repos made public specifically to get real branch protection.**
GitHub's classic branch protection API (`github_branch_protection`)
requires a paid plan for private repositories -- found live,
repeatedly, as `github_repository_environment.production` applied fine
but `github_branch_protection` then failed outright with "Upgrade to
GitHub Pro or make this repository public." Rather than pay for a plan
or go without real merge protection, `terraform-state`, `repo-infra`,
`k3s-bootstrap`, `secrets-manager`, and `k3s-apps` were each made
public, one at a time, only after confirming nothing in that specific
repo's content would be a real problem to expose:

- `terraform-state`/`repo-infra`/`k3s-bootstrap`: each is the account's
  own IAM/OIDC/RBAC control-plane code -- real, exposes the exact
  permission-boundary structure, but no actual credential (every access
  key stays in `pass`/PGP-encrypted or human-only, never in git), and
  each repo's own self-escalation design was built to hold up under
  scrutiny in the first place, not to depend on the policy shape
  staying secret.
- `secrets-manager`: only ever creates empty `aws_secretsmanager_secret`
  *containers* (no `GetSecretValue`/`PutSecretValue`/`DeleteSecret`
  anywhere in its Terraform) -- publishing it exposes secret
  names/descriptions that weren't public before (e.g. "Blocky's own
  Postgres query-log password"), never a real secret value.
- `k3s-apps`: real secret material is SOPS/Secrets-Manager-sourced, not
  committed -- publishing it exposes real internal LAN topology (IPs,
  NFS paths, node names) that wasn't public before, judged an
  acceptable tradeoff for a homelab-scale cluster.

No repo in this workspace has ever had a real credential committed to
its git history; that invariant is what makes each of these
public-repo decisions defensible on its own terms rather than a single
blanket policy.

## Three-stage rollout: trivy config, then trivy image, then gitleaks

The audit this ADR documents was itself staged: `trivy-config` (this
document's own subject) first, then `github/gha-common`'s
`trivy-image.yml` (container image CVE scanning), then its
`gitleaks.yml` (git-history secret scanning) -- each its own reusable
workflow, added and rolled out to every caller one at a time.

**Gitleaks, not `trivy fs`, for the git-history scan.** Confirmed
live: `trivy fs` only scans the current checked-out tree, not git
history -- it found nothing in a test repo with a secret committed and
then later removed in a following commit, while gitleaks found it
immediately. Since the whole point of this stage is catching a secret
that was committed and later removed, `trivy fs` would have been a
hollow checkbox for exactly that case.

## Addendum 2026-10-10: `home-infra-docs` to be made public

Decided by Julian 2026-10-10. Since `home-infra-ai-context` was merged in
(home-infra-docs#8), this is the single docs repo, changed through PRs and
gated by gitleaks, but `main` had no branch protection. Same tradeoff as
above: public for real protection instead of a paid plan. Change:
`github/repo-infra` `config.yml` sets `visibility: public` and requires
`gitleaks / Gitleaks secret scan` (repo-infra#40); local apply by Julian.

Content audit of every commit reachable from GitHub, PR refs included
(129 commits): no credentials, private keys, tokens, AWS account ID,
public IPv4/IPv6 addresses or personal data (address, birthday, phone,
bank). Two identifiers that do appear were already public:
`julian.kandler@outlook.com` (www.jkandler.de, `k3s-apps`) and the
`Finanzfluss_Nextcloud.xlsx` workbook name (`k3s-apps`). Exposed for the
first time: the LAN topology and known-gaps list in `current-state.md`,
the same class of exposure accepted for `k3s-apps` above.

One residual: `refs/pull/8/head` (commit `5ecc6e6`) carries the full IDs
of two access keys of the IAM user `julian` (`main` shows only the last
four characters). Both keys are deleted (the inactive one by Julian on
2026-10-10, before the switch), so both IDs are inert; the secrets were
never committed. PR refs can only be removed by GitHub support.
