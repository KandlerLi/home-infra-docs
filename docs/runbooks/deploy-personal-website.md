# Runbook: Deploy the personal website

Status: **Implemented, not yet applied or pushed.** See
[ADR 0016](../adr/0016-personal-website.md) for the design. This deploys a
static site (`infra/website`) to `https://www.jkandler.de/` via S3 +
CloudFront + ACM, following the same Terraform + GitHub Actions OIDC
pattern `dyndns` established. `jkandler.de` (bare) is untouched by this —
it stays owned by `dyndns`'s DynDNS record.

Every step below needs a human-selected, short-lived AWS and/or GitHub
identity. None of it can be run by an agent session.

## 1. Push the repository to GitHub

```bash
cd /home/julian/projects/infra/website
gh repo create KandlerLi/website --public --source=. --remote=origin
git push -u origin main
```

If you'd rather let `repo-infra`'s apply create the GitHub repository
first (step 2), just add the remote and push afterward instead.

## 2. Apply `repo-infra`

Creates the `website` GitHub repository's settings/branch protection and
the AWS OIDC plan/apply roles scoped to exactly what this deployment
needs (S3 bucket, CloudFront, ACM, and the existing `jkandler.de` Route53
zone — see `aws_policies.tf`'s `website` entry).

```bash
cd /home/julian/projects/bootstrap/repo-infra
terraform plan
terraform apply
```

## 3. Set the GitHub repository variables

Under **Settings → Secrets and variables → Actions → Variables** on
`KandlerLi/website`:

| Repository variable | Value |
|---|---|
| `AWS_ROLE_ARN` | The `website-github-actions` role ARN from the `repo-infra` apply output |
| `AWS_PLAN_ROLE_ARN` | The `website-github-plan` role ARN from the same output |
| `AWS_ACCOUNT_ID` | The AWS account ID |
| `ROUTE53_ZONE_ID` | The same hosted-zone ID used by `dyndns`, e.g. `Z07879811I86VC8PAL8HX` |

## 4. Trigger the first apply

Push to `main` (or re-run the `Apply` workflow via `workflow_dispatch`).
This runs `terraform apply` — creating the S3 bucket, CloudFront
distribution, ACM certificate, its DNS validation records, and the
`www.jkandler.de` alias records — then `aws s3 sync site/` and a
CloudFront cache invalidation.

ACM DNS validation can take a few minutes; `aws_acm_certificate_validation`
blocks the apply until the certificate is actually issued, so a longer
first-run duration than usual is expected.

## 5. Verify live

1. `https://www.jkandler.de/` resolves, serves the placeholder page, and
   has a valid certificate (check the padlock/certificate chain, not just
   that the page loads).
2. `https://jkandler.de/` (bare) is completely unaffected — still resolves
   to the home network exactly as before.
3. Open a pull request against `website` and confirm CI produces a clean
   speculative `terraform plan` with no unexpected diff.
4. Edit `site/index.html`, push to `main`, and confirm the change appears
   at `https://www.jkandler.de/` within a minute or two (CloudFront
   invalidation propagation).

## 6. Roll back

`terraform destroy` in `infra/website` removes the S3 bucket, CloudFront
distribution, ACM certificate, and DNS records. CloudFront distribution
deletion can take several minutes (it must first be disabled). This does
not affect `dyndns` or the `jkandler.de` apex in any way.
