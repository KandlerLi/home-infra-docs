# ADR 0016: Personal website at www.jkandler.de on S3 + CloudFront

- Status: Accepted
- Date: 2026-08-22
- Deployment status: Deployed, confirmed live at `https://www.jkandler.de/`

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
new Terraform root ([`website`](https://github.com/KandlerLi/website)) using the same shared
`jkandler-terraform-state` S3 backend, GitHub repository and AWS deploy
roles provisioned by [`repo-infra`](https://github.com/KandlerLi/repo-infra) (`config.yml` +
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

**GitHub-hosted runners, not the self-hosted home runner — superseded, see
the 2026-08-23 Update below.** `dyndns` runs its CI on `[self-hosted, home,
debian]`. The `website` repo originally deliberately did not: the entire
point of putting the CV on AWS was to stop depending on the home
network/server for it to be *reachable*. Tying its *deploy pipeline* back
to the home runner would partially undo that — a content fix couldn't
ship while the homeserver was down. The job only needs Terraform and AWS
OIDC, neither of which needs home-network access, so `runs-on:
ubuntu-latest` was strictly better here. This also meant no changes to
`home-infra`'s `github_runner` role were needed for this repo — until the
user explicitly asked to switch anyway, discussed below.

## Consequences

- `https://jkandler.de/` (bare) is unaffected by this decision at the DNS
  level and continues to resolve to the home network exactly as before —
  see the Update below for what Traefik now does with that traffic.
- The `website` repository's own README documents the one thing nobody
  can automate: setting `AWS_ROLE_ARN`/`AWS_PLAN_ROLE_ARN`/
  `AWS_ACCOUNT_ID`/`ROUTE53_ZONE_ID` as GitHub repository variables after
  the first `repo-infra` apply creates the real role ARNs. (Superseded in
  practice: `repo-infra`'s `modules/repo` already sets these automatically
  — no manual step was actually needed once `repo-infra` was applied.)

## Update (2026-08-23): apex redirect via Traefik, not DNS

Deploying the site surfaced a second, unrelated finding: a completely
untracked static site stack (S3 + CloudFront + wildcard ACM cert) already
existed for the bare `jkandler.de` apex, dating to 2024-02-12 — see
`handovers/2026-08-23-decommission-legacy-whoami-site.md` in
`home-infra-ai-context`. It was decommissioned; unrelated to the decision
below.

Once `www.jkandler.de` was live, the bare apex still 404'd for any visitor
who left off `www` (confirmed live: `curl -k https://jkandler.de/` returned
Traefik's own `404 page not found`, i.e. no router matched — not a bug,
exactly the consequence of the original decision above). Rather than leave
that as dead weight or reopen the apex-migration option this ADR explicitly
avoided, `home-infra`'s `shared_ingress` role gained an opt-in
`shared_ingress_apex_redirect_enabled` flag: a Traefik router + its own
Let's Encrypt certificate for `jkandler.de`, redirecting (301, via
`redirectRegex`) to `https://www.jkandler.de/`. This needed **no DNS or AWS
change at all** — the apex A record stays exactly as `dyndns`'s Lambda
manages it; Traefik simply now answers for that hostname instead of
falling through to its default 404, the same way it already answers for
`nextcloud.jkandler.de`/`ai.jkandler.de`/`torrent.jkandler.de`.

## Update (2026-08-23): switched to the self-hosted runner after all

While setting up self-hosted-runner CI/CD for a new `aws-budget` repo, the
user hit friction from the fine-grained GitHub PAT (`github_runner_github_token`)
being scoped only to `dyndns` — the only repo that had ever needed runner
registration. Working through that surfaced a question worth asking
directly: did `website` actually need to stay off the self-hosted runner?

The user was shown the tradeoff explicitly — a homeserver outage would
newly block shipping a site fix, something that wasn't true before, even
though the *deployed* site itself stays reachable regardless (CloudFront/S3
don't care where the CI ran) — and chose to accept it, asking for
consistency across all repos. `website`'s `checks.yml`/`apply.yml` moved to
`[self-hosted, home, debian]`, `repo-infra`'s `config.yml` gained
`runner: true` for it, and `home-infra`'s own checks (which have no
availability tradeoff either way, since they never touch the homeserver
regardless of where they run) moved to the self-hosted runner too, for the
same consistency reason.

This is recorded as an explicit, informed reversal, not a silent drift:
the original reasoning above was sound and is still accurate as a
description of the tradeoff — the user simply weighed it differently once
asked directly, preferring one consistent CI setup over the marginal
resilience benefit for this particular low-traffic personal site.
