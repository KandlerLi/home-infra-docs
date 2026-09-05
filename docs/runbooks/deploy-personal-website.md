# Runbook: Deploy the personal website

Status: **Deployed, confirmed live at `https://www.jkandler.de/`.** See
[ADR 0016](../adr/0016-personal-website.md) for the design. This deploys a
static site (`aws/website`) to `https://www.jkandler.de/` via S3 +
CloudFront + ACM, following the same Terraform + GitHub Actions OIDC
pattern `dyndns` established. `jkandler.de` (bare) is untouched by this —
it stays owned by `dyndns`'s DynDNS record.

Every step below needs a human-selected, short-lived AWS and/or GitHub
identity. None of it can be run by an agent session.

## 1. Apply `repo-infra`

Creates the `website` GitHub repository itself (via the `github_repository`
Terraform resource — do not create it imperatively with `gh repo create`
or the GitHub UI), its settings/branch protection, the AWS OIDC plan/apply
roles scoped to exactly what this deployment needs (S3 bucket, CloudFront,
ACM, and the existing `jkandler.de` Route53 zone — see `aws_policies.tf`'s
`website` entry), **and** the `AWS_ROLE_ARN`/`AWS_PLAN_ROLE_ARN`/
`AWS_ACCOUNT_ID`/`ROUTE53_ZONE_ID` GitHub Actions repository variables —
`modules/repo` wires the first three straight to the real role ARNs it
just created, and `ROUTE53_ZONE_ID` comes from `website`'s
`action_variables` entry in `config.yml`. No manual variable-setting step
is needed; if you ever find yourself pasting a value into the GitHub UI
here, something has gone wrong with the Terraform instead.

```bash
cd /home/julian/projects/bootstrap/repo-infra
terraform plan
terraform apply
```

## 2. Push the repository content

The repo now exists on GitHub but is empty. Push the local commit:

```bash
cd /home/julian/projects/aws/website
git remote add origin git@github.com:KandlerLi/website.git
git push -u origin main
```

## 3. Trigger the first apply

Push to `main` (or re-run the `Apply` workflow via `workflow_dispatch`).
This runs `terraform apply` — creating the S3 bucket, CloudFront
distribution, ACM certificate, its DNS validation records, and the
`www.jkandler.de` alias records — then `aws s3 sync site/` and a
CloudFront cache invalidation.

ACM DNS validation can take a few minutes; `aws_acm_certificate_validation`
blocks the apply until the certificate is actually issued, so a longer
first-run duration than usual is expected.

## 4. Verify live

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

## 5. Roll back

`terraform destroy` in `aws/website` removes the S3 bucket, CloudFront
distribution, ACM certificate, and DNS records. CloudFront distribution
deletion can take several minutes (it must first be disabled). This does
not affect `dyndns` or the `jkandler.de` apex in any way.
