# ADR 0016: Personal website at www.jkandler.de on S3 + CloudFront

- Status: Accepted
- Date: 2026-08-22
- Deployment status: Implemented, not yet deployed

## Context

The user wants a minimal personal website (CV/projects to follow later; an
MVP for now) reachable via the `jkandler.de` domain. Two open questions:
what to build it with, and where/how to host it.

Investigating the domain turned up a real conflict. The apex `jkandler.de`
A record is already live infrastructure: the `dyndns` repository's Lambda
keeps it pointed at the user's home IP via FRITZ!Box DynDNS updates, and
every home service (`nextcloud.jkandler.de`, `torrent.jkandler.de`, etc.)
is a CNAME onto that same apex record (see `dyndns`'s own README and
`main.tf`). Putting the CV site on the bare `jkandler.de` would mean
migrating that A record away from home-network duty — a materially bigger
and riskier change than the user was asking for.

## Decisions

**`www.jkandler.de`, not the apex.** Confirmed with the user via
AskUserQuestion. A new record, zero risk to the existing DynDNS/home
setup. `jkandler.de` bare keeps resolving to the home network exactly as
it does today and will not show the CV; there is currently no apex
redirect to `www`, since adding one would need to touch the DynDNS-owned
apex record.

**Plain HTML5/CSS3, no framework, no build step.** The user asked whether
Angular or plain HTML/CSS was the better call. There is no interactivity
to justify a framework for a CV/projects page, and static files are what
makes the cheap, low-maintenance hosting below possible in the first
place.

**S3 + CloudFront + ACM, following the established Terraform + GitHub
Actions OIDC pattern.** The user's own instinct was AWS S3; this repeats
the shape `dyndns` already established rather than inventing a new one: a
new Terraform root (`infra/website`) using the same shared
`jkandler-terraform-state` S3 backend, GitHub repository and AWS deploy
roles provisioned by `bootstrap/repo-infra` (`config.yml` +
`aws_policies.tf`), OIDC-based short-lived AWS credentials, no IAM users
or long-lived keys. The S3 bucket is private with no static-website-
hosting endpoint or public bucket policy; CloudFront reaches it only
through Origin Access Control. The ACM certificate is requested in
`us-east-1` (a second aliased AWS provider), since CloudFront requires
certificates there regardless of where the bucket itself lives.

**Content deployed via `aws s3 sync` in CI, not Terraform-managed
`aws_s3_object` resources.** Terraform manages the infrastructure (bucket,
distribution, certificate, DNS); the `apply.yml` workflow runs `aws s3
sync site/ s3://<bucket> --delete` plus a CloudFront invalidation right
after `terraform apply`, using the same OIDC role Terraform just assumed.
This keeps content changes fully within version control and CI — nothing
manual, nothing an imperative one-off against production — while avoiding
the friction of tracking every content file (including future binary
assets like a CV PDF or a photo) through Terraform state hashes, which
buys little for what is fundamentally "publish the current contents of
`site/`".

**GitHub-hosted runners, not the self-hosted home runner.** `dyndns` runs
its CI on `[self-hosted, home, debian]`. The `website` repo deliberately
does not: the entire point of putting the CV on AWS was to stop depending
on the home network/server for it to be *reachable*. Tying its *deploy
pipeline* back to the home runner would partially undo that — a content
fix couldn't ship while the homeserver was down. The job only needs
Terraform and AWS OIDC, neither of which needs home-network access, so
`runs-on: ubuntu-latest` is strictly better here. This also means no
changes to `home-infra`'s `github_runner` role were needed for this repo.

## Consequences

- `https://jkandler.de/` (bare) is unaffected by this change and continues
  to serve home-network traffic exactly as before.
- Only `https://www.jkandler.de/` will show the site. If a bare-apex
  redirect to `www` is wanted later, it needs a deliberate follow-up
  decision about what happens to the DynDNS-owned apex record — not
  something to retrofit casually.
- The `website` repository's own README documents the one thing nobody
  can automate: setting `AWS_ROLE_ARN`/`AWS_PLAN_ROLE_ARN`/
  `AWS_ACCOUNT_ID`/`ROUTE53_ZONE_ID` as GitHub repository variables after
  the first `repo-infra` apply creates the real role ARNs.
