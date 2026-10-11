# Decisions Log

Decisions that have no ADR of their own, plus context code comments point
at. Decisions with an ADR live only in [`adr/`](adr/README.md); sections
that merely retold an ADR were removed 2026-10-11 (see git history).

## Cloud inference first

The i5-2400 and the lack of a GPU still make a useful primary local
LLM an unreasonable baseline. The homeserver orchestrates a cloud model. Ollama
may be added later as an optional provider, never as a hidden prerequisite.

## Infrastructure-managed DNS

Stable subdomain CNAMEs belong in [`dyndns`](https://github.com/KandlerLi/dyndns), not as manual
Route 53 console changes. GitHub Actions uses OIDC and protected environments;
the FRITZ!Box DynDNS secret value is outside Terraform state.

## No public IPv6 without proof

A previous AAAA record sent traffic to the FRITZ!Box and exposed its
self-signed certificate. Public AAAA remains prohibited until address updates,
firewall, listeners, hostname routing, and TLS are verified externally.

## Human approval before writes

Future Git/Ansible tooling may prepare changes, run validation, and show diffs.
It must not deploy automatically. Nextcloud knowledge retrieval is a separate,
opt-in privacy decision.

## Infrastructure changes always go through IaC, never imperative commands

Live infrastructure state must only be changed by editing the owning
Terraform or Ansible source and applying it — never by running an imperative
one-off command (`gh api`/`gh` CLI, `aws` CLI, a shell script wrapping
either, or a manual web UI edit) against the live system, even for a change
that looks small enough to not bother writing as code.

This was made explicit after a live audit of `dyndns`'s branch protection
found exactly the failure mode it prevents: the repository's real branch
protection (required status checks, required approving review count,
signature/force-push/deletion rules, admin enforcement) is entirely
configured by `dyndns/scripts/protect-repository.sh`, an imperative script
that calls the GitHub API directly. `repo-infra`'s Terraform-managed
`github_repository_ruleset` exists alongside it but is `enforcement =
"disabled"` — a no-op decoy. `terraform plan` in `repo-infra` cannot detect
drift in the protection that is actually live, because it was never told
that protection exists. A change made by hand or by re-running the script is
invisible to the IaC that is supposed to be this workspace's single source
of truth. Bringing branch protection under actual Terraform management (see
ADR 0009) is the concrete fix for this specific gap; this decision
is the general rule that prevents new gaps of the same shape.

## dyndns merges no longer require a separate owner-approval status

`dyndns`'s `owner-approval.yml` workflow (an exact-commit status-check gate
requiring an explicit `/approve` comment before merge, working around
GitHub's rule that a sole author can't approve their own PR) was removed on
2026-08-21, and `"Owner approval"` dropped from `repo-infra`'s
`required_status_check_contexts` for `dyndns` (via `terraform apply`, `0
added, 1 changed, 0 destroyed`, verified live). The reasoning: since
`pull_request_creation_policy: collaborators_only` plus a single collaborator
(the owner) already make it structurally impossible for anyone but the
owner to merge — merging requires GitHub write access, which only the owner
has — the extra manual gate protected only against the owner's own
mistakes, not against anyone else. The `production` environment's reviewer
gate (approval required before the apply job receives AWS credentials)
already provides an equivalent manual checkpoint, closer to the point of
actual risk (AWS changes) than the merge itself.

The desired workflow going forward: open a PR, review the `Terraform plan`
job's speculative plan output in the checks tab, merge once `Validate` and
`Terraform plan` are green (no separate approval step), then approve the
production environment deployment before Terraform applies. `CODEOWNERS`
keeps naming `@KandlerLi` as sole code owner for documentation, with its
stale explanation of the removed mechanism trimmed; `require_code_owner_reviews`
remains `false`, so it was never functionally enforced.

## The production environment's reviewer gate is gone too

Picked back up 2026-09-07: the "then approve the production environment
deployment" step this file's own previous entry described as the desired
workflow is gone now. The same reasoning that removed `owner-approval.yml`
on 2026-08-21 applies a second time, one layer later — an AI agent may
commit and open PRs in this workspace, but merging stays a deliberate human
action the agent never performs (the owner's standing rule), so the
separate "approve deployment" click was reviewing nothing a second pair of
eyes hadn't already seen at merge time.

Removed: the `reviewers` block on `github_repository_environment
"production"` in `repo-infra/modules/repo/main.tf` (one shared resource,
applied via `terraform apply`, uniformly to every repo with
`branch_protection_enabled` — `dyndns`, `website`, `aws-budget`,
`homeserver-health-check`, `ses-relay`, `k3s-apps`). Kept: the environment
resource itself, since it's also the AWS OIDC trust boundary
(`github_apply_oidc_subject` requires `repo:.../environment:production` in
the token) — deleting the whole resource instead would have broken AWS
access for all six repos, not just removed a redundant click.

Current flow: open a PR, review the `Terraform plan` job's speculative plan
output, merge once `Validate` and `Terraform plan` are green — Terraform
applies immediately, no separate approval step left.

## Self-hosted GitHub Actions runners are opted into via `config.yml`, wired by a generated variable

Adding an AWS-deploying repository already meant editing only `repo-infra`'s
`config.yml` and `aws_policies.tf`. Attaching a self-hosted GitHub Actions
runner to a repository was still a separate, manually-maintained list:
`home-infra`'s `ansible/inventory/group_vars/all/main.yml` hardcoded
`github_runner_github_repositories` independently of `repo-infra`, so adding
a runner-backed repository meant editing two unrelated files in two
repositories with no link between them.

Terraform's `hashicorp/github` provider cannot close this gap on its own —
the only runner-related resource it exposes is `github_actions_runner_group`,
an org/enterprise-level construct for routing which repos may use which
runner groups. There is no resource to obtain a runner registration token or
to actually attach a runner process to a repository; that is inherently
imperative (`POST .../actions/runners/registration-token` followed by
running `config.sh`/`run.sh` against the runner host), which is exactly what
`home-infra`'s existing `github_runner` Ansible role already does and must
keep doing.

The fix keeps `config.yml` as the single place a human decides which
repositories exist and what they get, without pulling runner provisioning
into Terraform: a repository entry may now set `runner: true` (a convention
key Terraform's `repo-infra` module never reads — untyped `yamldecode`
tolerates the extra key). `home-infra`'s
`scripts/sync_github_runner_repositories.py` reads `repo-infra`'s
`config.yml`, filters to `runner: true` entries, and regenerates
`ansible/inventory/group_vars/all/github_runner_repositories.yml` — a
dedicated, clearly-marked generated file, so hand-maintained variables in
`main.yml` are never silently overwritten. Adding a runner-backed repository
is now: add `runner: true` to `config.yml`, run the sync script in
`home-infra`, review and commit the regenerated file, run the
`github-runner.yml` playbook. Still two tools and two repositories — Ansible
provisioning was deliberately not folded into the Terraform apply, since
that would mean Terraform triggering long-running, imperative VM
configuration via `local-exec`, a much larger and riskier change for a
homelab-scale setup — but now driven from one source of truth instead of two
independently-maintained lists.

**Update 2026-10-11:** the runner VM and its Ansible role are gone; runners
are Deployments in [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap) (since 2026-08-31). The
decision stands, only the target changed: the same sync script now
writes [`k3s-bootstrap/repositories.auto.tfvars.json`](https://github.com/KandlerLi/k3s-bootstrap/blob/main/repositories.auto.tfvars.json). Adding a
runner-backed repository is now: `runner: true` in `config.yml`, run
`scripts/sync_github_runner_repositories.py` in `home-infra`, commit the
regenerated file in `k3s-bootstrap`, and apply `k3s-bootstrap` locally.

## Secrets moved from SOPS/GPG to AWS Secrets Manager

An audit (raised as "Secrets sprawl: audit, document, and consolidate")
found 27 secrets spread across two SOPS vaults sharing one GPG key
(`home-infra`'s own `secrets.sops.yml`, 25 keys; `k3s-apps`'s own, 2
keys), reached through three different consumption mechanisms
depending on which repo needed which key: Ansible's `community.sops.sops`
vars plugin (4 keys, direct), `sops -d` piped into `TF_VAR_*` for
Terraform (the large majority, both local applies and `k3s-apps`' own
CI), and a separate export path for [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)'s
`github_runner_github_token` (human-apply only). The same audit
surfaced real drift this design had let accumulate silently: a dead
variable (`deluge_web_password`, no live consumer left), a broken
mapping in `sync_secrets_to_pass.py` (referencing an already-deleted
role file), and a GitHub Actions secret read by `k3s-apps`' CI but
never declared in `repo-infra/config.yml` at all.

The fix follows [`dyndns`](https://github.com/KandlerLi/dyndns)'s own existing pattern rather than
inventing a new one: Terraform creates the secret *container* only
([`terraform-state`](https://github.com/KandlerLi/terraform-state)'s `secrets_manager.tf`), never its value;
the value is set out-of-band via `aws secretsmanager put-secret-value`,
keeping real secret material out of Terraform state the same way SOPS
kept it out of plaintext git. Secrets are grouped one entry per logical
service (`home-infra/authelia`, `home-infra/grafana`, etc.), not one
per source file (too coarse, blurs an already-coarse trust boundary
further) and not one per individual key (matches `dyndns`'s own literal
precedent, but ~$10.80/month alone — roughly the whole prior $10/month
account-wide budget). One real, deliberate deviation from `dyndns`'s
own precedent: `k3s-apps`' plan-role IAM grant needs `GetSecretValue`
too, not just metadata reads, because `jsondecode()` at `terraform plan`
time needs the actual value to compute a diff, unlike `dyndns`'s
Lambda-runtime-only secret.

The cutover shipped in deliberately separated, independently-verified
phases across all five repos it touched
([`terraform-state`](https://github.com/KandlerLi/terraform-state)/`home-infra`/`k3s-apps`/`repo-infra`/
[`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)) rather than one large change, after two real
production outages earlier in the same effort (Authelia's own OIDC
secrets going empty) established the actual failure mode to guard
against: never bundle "grant new access" with "destroy old access" in
one apply if the old code path might still run in the gap before the
new one is verified. Concretely, `repo-infra`'s own GitHub Actions
secret-sync removal was held back as a separate apply until a real
`terraform plan` proved `k3s-apps`' CI no longer read any of the 21
secrets it was about to destroy — the same discipline applied to every
phase. `secrets.sops.yml`, `.sops.yaml`, and the `community.sops.sops`
vars plugin were only deleted once every real consumer had been cut
over and verified live for a stability window, not alongside the
consumers themselves.

## AWS Secrets Manager containers live in their own bootstrap repo

The SOPS-to-Secrets-Manager cutover left all 10 `aws_secretsmanager_secret`
container resources in [`terraform-state/secrets_manager.tf`](https://github.com/KandlerLi/terraform-state/blob/main/secrets_manager.tf).
They are being moved into a dedicated [`secrets-manager`](https://github.com/KandlerLi/secrets-manager) root,
one secret at a time, so the secrets concern is separable from the
state-bucket bootstrap concern and can carry its own per-secret
documentation and rotation tooling — without `terraform-state` (a
bootstrap-once root) accreting an ongoing-maintenance surface.
`secrets-manager` is a third bootstrap-category root beside
`terraform-state` and `k3s-bootstrap`: private mirror, no CI, applied
only locally as root — never as `julian`, since the containers it owns
are what `julian`'s Secrets Manager grant points at (the same
self-escalation reasoning as `terraform-state`'s `operator.tf`). Layout
is one self-contained module directory per secret
(`secrets/<repo>/<service>/`: its `main.tf` with the
`aws_secretsmanager_secret` resource + `arn` output, and its
`README.md`).

The move uses the ADR 0006 / ADR 0010 no-destroy handoff: a transient
`import` block in `secrets-manager` (targeting
`module.X.aws_secretsmanager_secret.this`, stripped after the first
apply per precedent), a permanent `removed { ... destroy = false }`
block left in `terraform-state`. `julian`'s IAM grant does **not**
follow the containers — it stays in `terraform-state/operator.tf` next
to the rest of `julian`'s operator policy, with containers still local
covered by resource references and migrated ones covered by a wildcard
ARN string (`...:secret:<name>-*`); switching one to the other is an
in-place policy edit with continuous ARN coverage, so there is no
access gap. (`k3s-bootstrap-local`'s read grant on
`home-infra/github-runner` works the same way when that one migrates.)
Consumers are unaffected: every one ([`k3s-apps`](https://github.com/KandlerLi/k3s-apps)'s `secrets.tf`,
[`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)'s, [`home-infra`](https://github.com/KandlerLi/home-infra)'s Ansible,
[`repo-infra`](https://github.com/KandlerLi/repo-infra)'s CI policy) reads secrets by name / ARN pattern,
never by which root owns the container.

Rotation automation was scoped honestly against the network reality:
an AWS Lambda has no route to the private k3s network or the
homeserver, and most of these secrets either have real blast radius
(Authelia's crypto material logs everyone out), no programmatic
mint API (GitHub PATs, Nextcloud app passwords), or a validator that
lives on the LAN. Only `blocky_postgres_password` gets an automated
rotation — a scheduled GitHub Actions workflow on the in-cluster
self-hosted runner (which already has `kubectl` and runs `terraform
apply`), not an AWS-native rotation Lambda. Everything else is
manual-by-design, with per-secret runbooks; where the mechanical
post-mint steps are worth scripting (e.g.
`k3s-apps/sankey-export`), the script lives in [`k3s-apps`](https://github.com/KandlerLi/k3s-apps)
alongside the apply it runs, not in `secrets-manager`.

`k3s-apps/sankey-export` was the pilot (migrated and value-rotated
end-to-end 2026-09-10); `home-infra/grafana` followed the same day,
with `monitoring_grafana_admin_password` *removed* in the same pass
(Grafana's native login is disabled at the protocol level, so it could
not be used to log in -- the k3s copy now generates a throwaway
`random_password`, and the `pass`/Ansible copies went too).
`home-infra/open-webui` (2026-09-11) and `home-infra/home-agent`
(2026-09-11) followed with no bundled cleanup, each already holding
only live/needed keys. `home-infra/ingress` (2026-09-11) had a real
bundled cleanup: `shared_ingress_auth_password`/`_hash` were retired
outright rather than migrated — the `shared-auth` Traefik Basic Auth
rollback they fed had sat unreferenced since the 2026-09-08 Authelia
forward-auth cutover, and stayed only as a hedge against Authelia's
own SQLite storage bug (then tracked in the private task list, closed 2026-09-16) recurring; once
forward-auth held stable for several more days with no recurrence,
the rollback stopped earning its keep. `home-infra/authelia` is
deliberately last.
[`secrets-manager/README.md`](https://github.com/KandlerLi/secrets-manager/blob/main/README.md) carries the live migration-status
table.

`k3s_ingress_acme_dns01_*`'s value was also rotated end-to-end the same
day — the first real proof that a *Terraform-managed* AWS credential
(as opposed to the human-typed/API-minted values every other rotation
this campaign has covered) can be rotated safely here.
`aws_iam_access_key.acme_dns01` ([`dyndns`](https://github.com/KandlerLi/dyndns)) has no in-place
rotation, only destroy+recreate; `lifecycle { create_before_destroy =
true }` makes `terraform apply -replace=aws_iam_access_key.acme_dns01`
mint the new key before deleting the old one (IAM allows 2 keys per
user), so there's a real zero-credential-less-window guarantee instead
of hoping the destroy-then-create gap doesn't collide with a live
renewal. This is the pattern to reach for any future Terraform-managed
credential rotation in this workspace, not a one-off for this key.
Also found live: `env.valueFrom.secretKeyRef` (unlike a sub_path
volume mount, which at least the Terraform-managed ConfigMaps already
had a checksum-annotation fix for) is read once at container start and
never refreshed at all — the Secret updated correctly but the running
Traefik Pod kept the just-deleted key in memory until a manual restart,
fixed with a `checksum/acme-dns01-credentials` annotation
([`k3s-apps`](https://github.com/KandlerLi/k3s-apps) PR #42, merged).

`home-infra/monitoring` (2026-09-11) is the one group in this campaign
with a real dual consumer — [`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' Terraform (Alertmanager
and Authelia both read it) *and* [`home-infra`](https://github.com/KandlerLi/home-infra)'s own Ansible
(`site.yml`'s `pre_tasks`, a `lookup('amazon.aws.secretsmanager_secret',
...)` call gated on `monitoring_enabled`). Verified both independently
rather than trusting the Terraform-only pattern every other migration
could rely on: a clean `k3s-apps` plan, and the exact same lookup call
run directly (a local control-node lookup, no `become`/remote host
needed) against the migrated container.

Its value was also rotated end-to-end the same day, second proof (after
the ACME key) that `create_before_destroy` + `terraform apply -replace`
is the right pattern for any Terraform-managed credential here — this
one adds a derive-script step (`[`ses-relay/scripts`](https://github.com/KandlerLi/ses-relay/blob/main/scripts)
derive_smtp_password.py`, AWS's own SigV4 SMTP-password algorithm) and
two consumers (Alertmanager, Authelia) instead of one, both updated by
a single `terraform apply` once `checksum/config` existed on
Alertmanager's own Deployment too (added proactively this time, before
finding the gap live, unlike the ACME rotation). Also corrected a real
inaccuracy found live: [`home-infra`](https://github.com/KandlerLi/home-infra)'s `site.yml` doesn't need a
re-run on rotation — its `pre_tasks` only assert these values are
non-empty, never template them anywhere (Alertmanager moved to k3s
outright).

`home-infra/nextcloud` (2026-09-11) followed, the first genuinely
Ansible-only consumer migrated (no Terraform side at all — Nextcloud
AIO stays on the homeserver). Verified the same way `monitoring`'s
Ansible side was: the exact lookup call, run directly, not just a
syntax-check.

Its OIDC pair was also rotated the same day — the third this session
(after Grafana/Open WebUI) and the first that genuinely needs
[`home-infra`](https://github.com/KandlerLi/home-infra)'s Ansible side, confirmed live by reading
`ansible/roles/nextcloud_aio/tasks/oidc.yml` directly (a real `occ
user_oidc:provider --clientsecret=...` upsert, unlike
`home-infra/monitoring`'s assert-only SES creds). Rather than have the
fresh plaintext/hash pair pasted into chat, `[`k3s-apps/scripts`](https://github.com/KandlerLi/k3s-apps/blob/main/scripts)
rotate-oidc-client-secret.sh` (previously grafana/openwebui only)
gained `nextcloud` support — same silent-prompt/swap-detection
mechanics, plus a `site.yml --ask-become-pass` re-run the script now
does itself after Authelia's own `terraform apply`. This is the
template for any future "Ansible-only consumer needs a coordinated
secret rotation" case: extend the existing script rather than ask for
secrets in the conversation.

`home-infra/github-runner` (2026-09-12) closed out the last group with
a non-`julian` grant: `k3s-bootstrap-local` (the scripted,
non-interactive identity that applies [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)) had
its own real resource-reference grant on this secret
(`ReadGithubRunnerSecret`), which needed the same wildcard-ARN swap as
`julian`'s own — a second identity to update, not just a second
statement. Verified as that identity for real (not `julian`'s own
credentials): [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)'s own `scripts/roll-out.sh`,
which sources `k3s-bootstrap-local`'s credentials from `pass`, showed
a real `Read complete` on the migrated secret and zero plan diff. Only
`home-infra/authelia` (deliberately last) and `home-infra/blocky`
(bundled with its own rotation-automation build) remain.

`github_runner_github_token` was also rotated end-to-end the same day.
Found live ahead of the rotation, not during it this time:
`modules/github_runner` had the exact same read-once-at-start
`secretKeyRef` gap the ACME/Alertmanager rotations already surfaced --
fixed proactively with a `checksum/token` annotation across all 7
Deployments before ever touching the real PAT. Real functional proof
came from GitHub's own runner API (`gh api
repos/<org>/<repo>/actions/runners`), not just a clean Terraform apply
-- confirming all 7 runners genuinely re-registered `online`. A
momentary "session already exists" registration race during the
checksum rollout (7 Deployments restarting near-simultaneously,
fighting over fixed `RUNNER_NAME`s) self-healed within a few container
restarts each; worth remembering if a future rotation of this same
group looks briefly rocky right after applying -- it settles on its
own.

`dyndns/fritzbox` (2026-09-12) extended the campaign beyond its
original 10-group inventory (requested directly) and beyond
[`terraform-state`](https://github.com/KandlerLi/terraform-state) as a source root for the first time: this
container was created directly in [`dyndns`](https://github.com/KandlerLi/dyndns), a real CI/PR-gated
repo. Confirmed live before migrating that `julian` had no explicit
grant on it at all — every read this session had used root. The
`removed`/`import` no-destroy handoff generalizes cleanly to a
non-bootstrap source root; the one real difference is that
"source relinquishes" needs a normal PR there instead of a direct
local apply, and that PR's `removed` block then lives in that repo
permanently rather than in [`terraform-state`](https://github.com/KandlerLi/terraform-state) — it doesn't
count toward that repo's own final-sweep condition. Also fixed in the
same PR: the Lambda's own `CREDENTIALS_SECRET_ID` env var switched
from an ARN reference to the secret's plain name (boto3's
`get_secret_value` resolves either equally well), removing the need to
reconstruct an ARN this root no longer has a local resource for.

Its value was also rotated the same day. Unlike every other rotation
this campaign, the credential itself isn't consumed by anything
Terraform/Ansible/kubectl reaches — it's plain HTTP Basic Auth the
FRITZ!Box router sends on its own, so the router's own admin UI needed
reconfiguring by hand, the one step in this whole campaign nothing but
a human at a physical device can do. Verified for real with a direct
HTTPS request to the live update endpoint using the new credentials
and the DNS record's actual current IP — an UPSERT to the
already-correct value, a safe test with no risk of changing what's
served, succeeded on the first try (no need to wait out the Lambda's
own 5-minute credential cache).

`home-infra/blocky` (2026-09-12) closed out all but one of the
original 10-group inventory, done specifically because Julian
confirmed Blocky isn't currently anyone's live DNS resolver -- running
but unused, a genuinely low-risk window. Confirmed live before
migrating: both consumers (`modules/blocky`, `modules/grafana`) had
zero checksum annotations between them for `blocky_postgres_password`
-- fixed first, since the rotation-automation build this migration
exists to unblock would otherwise silently not restart either Pod on
a real rotation. Only `home-infra/authelia` remains.

Blocky rotation automation (increment 2 of this same campaign) was
built and proven live the same day. The key design decision: no new
`pods/exec` RBAC anywhere, even though the obvious way to run `ALTER
USER blocky ...` looks like `kubectl exec` into the live Pod. RBAC
can't scope `pods/exec` to one named Pod in a way that survives a
rollout (resource names churn on every ReplicaSet), so that path would
have meant granting the CI ServiceAccount exec into *any* Pod in the
namespace -- a real, hard-to-scope privilege escalation in a workspace
that has otherwise kept every CI grant narrow. Instead: the self-hosted
runner already lives inside the cluster network (it already runs
`terraform apply` against the in-cluster Kubernetes API), so the
rotation script just connects to `blocky-svc:5432` directly over TCP
-- the exact same in-cluster Service `modules/grafana`'s own datasource
config already reaches for its own dashboards. Zero new Kubernetes
RBAC; the only new grant needed was apply-only `secretsmanager:
PutSecretValue` on `home-infra/blocky-*` ([`repo-infra`](https://github.com/KandlerLi/repo-infra)), the
CI role's first-ever write access to any secret, deliberately its own
statement so a compromised token's write blast radius stays exactly
one secret. This is the template for any future "CI needs to touch a
live in-cluster service" automation here: reach it over the network via
its own Service, don't reach for `pods/exec`.

Verified with a real manual `workflow_dispatch` run rather than
trusting the code review alone: `ALTER ROLE` succeeded, `terraform
apply` rolled both Secrets and both Deployments in one pass (`0 added,
5 changed, 0 destroyed`), the new password authenticated against the
live database afterward, and both `blocky-svc`/`grafana-svc` responded
healthy -- the script's own verification steps are written to fail the
whole job loudly on any of these, not report a false success.

`home_agent_ghcr_token` was split out of `home-infra/home-agent` into
its own dedicated `k3s-apps/ghcr-pull-token` secret (2026-09-12) --
the one group so far that's a genuinely new container, not a
migration. It was never really "home-agent's own" credential: a
single, account-scoped GitHub PAT (`read:packages` on `KandlerLi`)
that `modules/home_agent` and `modules/sankey_export` both read
verbatim to pull two unrelated private packages
(`ghcr.io/kandlerli/home-agent`, `ghcr.io/kandlerli/sankey-export`).
Splitting it does *not* reduce actual coupling or blast radius --
both consumers still roll together on any rotation of this shared
value, regardless of which container holds it -- the benefit is purely
naming and discoverability: a reader of `home-infra/home-agent`'s own
README no longer sees a credential that isn't really about home-agent
at all, and `k3s-apps/ghcr-pull-token`'s own name and README now say
plainly what it actually is and who shares it.

Sequenced to avoid any gap or rotation risk: (1) create the new empty
container (real `1 to add`, no `import` block needed -- this value
never had its own resource before); (2) copy the *current*,
still-valid token value into it verbatim, a pure relocation with zero
functional change since the byte value is identical; (3) rewire both
Terraform consumers ([`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' `secrets.tf` + `main.tf`) to
read from the new location; (4) apply, expecting -- and getting -- a
literal zero diff on both `kubernetes_secret_v1.ghcr_pull` resources,
confirmed both by the Terraform plan itself and by decoding the live
Kubernetes Secret and comparing it byte-for-byte to the value staged
in step 2; (5) only after that was confirmed live and both consumers
(`home_agent`, `sankey_export`) stayed healthy, strip the now-dead key
out of `home-infra/home-agent`'s own JSON, confirmed by a follow-up
`terraform plan` showing "No changes" -- proof the module genuinely no
longer reads that key at all, not just that the code says so. The
actual token *rotation* is deliberately a separate, later step against
the newly-relocated secret, not bundled into this move.

One real gap this surfaced: [`repo-infra/aws_policies.tf`](https://github.com/KandlerLi/repo-infra/blob/main/aws_policies.tf)'s
`k3s_apps_secretsmanager_read_statements` grants read access to every
Secrets Manager group [`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' own Terraform reads directly
by name -- creating a *new* group there needs a matching new grant
just as much as migrating an existing one does, and this was missed on
the first pass. The PR's own `Terraform plan` CI check caught it
immediately and unambiguously (`AccessDeniedException: ... is not
authorized to perform: secretsmanager:GetSecretValue`) rather than
failing silently or producing a wrong plan -- exactly the kind of
real, CI-enforced check this campaign's `removed`/`import` discipline
exists to lean on. Fixed by adding the new ARN to that shared
`locals` list (covering both the plan and apply CI roles at once,
since [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) needs `GetSecretValue` even to compute a plan
diff) and re-running the check.

`home-infra/authelia` (2026-09-12) closed out the campaign: the last
of the original 10 groups, deliberately saved for last since it holds
9 keys and the highest blast radius of anything migrated (session
cookies and OIDC signing material whose rotation logs out every active
session across three services at once). The migration itself followed
the by-then well-worn procedure exactly, no new wrinkle: imported
clean (`1 to import, 0 to add/change/destroy` on the real resource
itself -- the `recovery_window_in_days` "change" every single import
in this campaign shows is a benign quirk of the AWS API, which never
returns that field on read since it only matters at delete time, not
a real drift), `julian`'s wildcard grant added additively before the
old resource-reference grant was dropped (no access gap either side),
and both [`secrets-manager`](https://github.com/KandlerLi/secrets-manager) and [`terraform-state`](https://github.com/KandlerLi/terraform-state)
confirmed clean afterward. [`k3s-apps`](https://github.com/KandlerLi/k3s-apps)'s own plan showed zero
diff throughout, as expected -- Authelia's consumer code never
referenced which repo owns the container, only the secret's name.

With every group migrated, the final sweep landed the same day:
[`terraform-state/secrets_manager.tf`](https://github.com/KandlerLi/terraform-state/blob/main/secrets_manager.tf) -- by then containing
nothing but ten `removed { destroy = false }` blocks and the `locals`
entry only those blocks' now-deleted `resource` blocks had ever
referenced -- was deleted outright, confirmed with a "No changes"
plan. `operator.tf`'s `ManageSecretsManagerSecrets` statement was
deliberately left untouched and stays permanently: it was never really
about which repo owns a container, only about `julian` needing
read/write on the secret values themselves, a grant that was always
going to outlive the migration regardless of where each container
ended up. Collapsing the ten wildcard ARN strings there into two
prefix patterns (`home-infra/*`, `k3s-apps/*`) was considered and
declined -- the per-secret entries read as an explicit, auditable list
of exactly what `julian` can touch, which is worth more than the few
saved lines.

One loose end surfaced along the way, not fixed as part of this
campaign: two of the credentials this workspace still rotates by hand
are durable GitHub PATs with no scoped, short-lived alternative in
place yet (`k3s-apps/ghcr-pull-token`, `home-infra/github-runner`'s own
PAT) -- a GitHub App's own installation tokens would remove that
manual-rotation chore entirely rather than merely easing it, but
swapping either PAT for one needs its own investigation first (whether
an installation token can actually authenticate a GHCR pull or a
self-hosted runner registration the same way). Moved to the private task list
now that the migration campaign itself has closed out, rather than
carried forward in the private task list as active pull. Closed 2026-09-17: GitHub
App installation tokens can't authenticate GHCR pulls, see "GHCR image
pulls stay on a classic PAT" below.

`home-infra/authelia`'s first real rotation (2026-09-13, 6 of its 9
keys) established the pattern for re-encrypting a live SQLite database
in this workspace -- worth recording since nothing here had needed it
before. `authelia_storage_encryption_key` can't just be swapped in
Secrets Manager the way every other key in this campaign was: Authelia
itself has to re-encrypt the on-disk database with
`authelia storage encryption change-key` first, and that has to run
while nothing else is writing to the file. The chosen sequence: scale
the Deployment to 0 (a temporary, self-correcting `kubectl scale` --
Terraform's own declared `replicas = 1` restores it on the very next
apply, so this doesn't create the kind of undetected drift the
IaC-only rule exists to prevent, the same reasoning that already
covers a plain `kubectl rollout restart` elsewhere in this workspace),
run `change-key` from a throwaway Pod that mounts the exact same PVC
and config Secret the real Deployment uses (same image, no new RBAC,
no separate Job resource needed), and only then write the new key to
Secrets Manager and `terraform apply` -- Terraform's own restored
`replicas = 1` brings Authelia back up already matching the
re-encrypted database, no separate "turn it back on" step. The
verification this pattern leans on is `authelia storage schema-info`:
run once with the *old* key (from the not-yet-updated config Secret,
still mounted read-only) to confirm it now reads "invalid" -- proof
the re-encryption actually happened, not assumed from a clean exit
code -- and once with the *new* key passed explicitly to confirm it
reads "valid". Both checks are read-only against the database, safe to
run as many times as needed before committing to the real
`put-secret-value`.

Two mid-rotation mistakes, both caught before they mattered: the two
prepared commands (an interactive password-hash prompt needing the
live pod, and the storage-scale-down script) were handed over in the
wrong order by mistake -- the scale-down ran first, taking the pod
down before the password hash could be generated against it. Recovered
by standing up a second, minimal throwaway Pod (no PVC, no live config,
just the same image with a `sleep` command) purely to host the
interactive `authelia crypto hash generate argon2` prompt, rather than
needing the real Deployment back up early. Separately, after the
scale-down, the instinct to "just start it back up" was caught and
corrected: the database was by then already re-encrypted with the new
key while the live config Secret still held the old one, so restarting
at that point (before the new key reached Secrets Manager) would have
handed Authelia a database it couldn't decrypt -- the schema-info
verification above exists specifically to make that risk checkable
rather than assumed away.

[`secrets-manager`](https://github.com/KandlerLi/secrets-manager) gained real CI automation 2026-09-13,
correcting a categorization inherited at its creation rather than
actually justified for it. It was scaffolded alongside
[`terraform-state`](https://github.com/KandlerLi/terraform-state)/[`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap) as a third
"rare, admin-privileged, human-applied-only, never CI" bootstrap root
-- reasonable at the time, since the original plan for this repo
considered having it grant `julian`'s own Secrets Manager access
directly, which would have shared those two roots' genuine
self-escalation risk. That plan changed before anything was built:
`julian`'s grant stayed in `terraform-state/operator.tf` instead (this
repo's own README, "IAM grants live in terraform-state, not here"),
but the "never CI" categorization never got revisited once the reason
for it no longer applied. Asked directly whether the same automation
`k3s-apps` has could extend here, the actual risk turned out to be
much smaller than the category suggested: this repo never touches IAM
at all, only ever creates empty `aws_secretsmanager_secret` containers
-- no `GetSecretValue`, `PutSecretValue`, or `DeleteSecret` anywhere in
its Terraform. A CI role scoped to exactly `CreateSecret`/
`DescribeSecret` on the three name prefixes it already manages
(`home-infra/*`, `k3s-apps/*`, `dyndns/*`) can't read or write any
actual secret value, or grant itself or anything else broader access --
worst case a compromised run creates a bogus empty container.

Two real gaps surfaced while making the change, both fixed live rather
than found the hard way later:

1. **Classic branch protection needs a paid GitHub plan for private
   repositories** -- the exact same constraint `k3s-apps` already hit
   and resolved by going public (its own `config.yml` comment says so
   explicitly), rediscovered here because the connection wasn't made
   until the real `github_branch_protection` apply failed with
   "Upgrade to GitHub Pro or make this repository public." Fixed the
   same way: `secrets-manager` is now public too. No real secret
   material ever touches this repo's git history either way -- only
   secret names/descriptions become visible that weren't before (e.g.
   "Blocky's own Postgres query-log password"), a minor reconnaissance
   signal at most, not a credential leak.
2. **A fine-grained GitHub PAT's repository access list doesn't update
   itself.** `home-infra/github-runner`'s own `github_runner_github_token`
   is scoped to an explicit list of repositories ([`secrets-manager/secrets/home-infra/github-runner/README.md`](https://github.com/KandlerLi/secrets-manager/blob/main/secrets/home-infra/github-runner/README.md)
   already documented this, for the 7 repos onboarded before this one)
   -- adding `secrets-manager` as an 8th `runner: true` repo in
   `config.yml` doesn't extend that list automatically, since it's a
   GitHub-web-UI-only setting the PAT's own owner edits by hand, not
   something Terraform or the GitHub API manages here. The new runner
   Pod crash-looped on a 403 obtaining its registration token until
   Julian added `secrets-manager` to the token's repository access
   list directly on GitHub. Verified for real afterward with a direct
   API call from an already-healthy runner Pod (using its own
   `$ACCESS_TOKEN`, never exposed) against
   `POST /repos/KandlerLi/secrets-manager/actions/runners/registration-token`
   -- a real `201` with a token, not just a clean-looking retry -- before
   trusting the fix and restarting the new Pod.

Deliberately retired in the same change: `scripts/roll-out.sh`, the
root-applied local script this repo had before. Once CI covers the
routine apply path, this workspace's established convention (already
set by `k3s-apps`, which never had such a script once its own CI
landed, and by [`dyndns`](https://github.com/KandlerLi/dyndns)'s own `protect-repository.sh`, deleted
outright rather than kept as a documented manual-exception carve-out)
is not to keep a maintained script making a now-automated path
routine to run by hand. A genuine emergency can still fall back to
plain local `terraform init/plan/apply` as root -- just without a
script smoothing that over as an accepted everyday alternative to the
PR-reviewed path.

A third gap, this one in the IAM scoping itself rather than the
surrounding wiring: the PR's own first real `Terraform plan` run
failed with `AccessDeniedException` on `secretsmanager:GetResourcePolicy`
-- the AWS provider's `aws_secretsmanager_secret` read calls that
action on every refresh (checking whether a resource-based policy
exists, whether or not one does), a separate action neither
`DescribeSecret` nor this repo's own Terraform would suggest was
needed. Same category of gap ADR 0018 already named as the tradeoff of
hand-enumerating exact permissions instead of a broad managed policy
(SNS and AWS Budgets had already each hit their own version of "the
real API surface needs more than the obviously-named action"). Fixed
by adding it to both the plan and apply statements, re-run, merged.

**Verified live, end to end, 2026-09-13**: PR #1 -- this repo's first
ever pull request -- passed real CI (`Build CI image`, `Validate`,
`Terraform plan` all green using the new `secrets-manager-github-plan`
role) and merged; the post-merge `apply.yml` run completed
`Apply complete! Resources: 0 added, 0 changed, 0 destroyed` using the
new `secrets-manager-github-actions` role -- proof the CI-driven apply
path works end to end without disturbing any of the real secret
containers already managed here. The runner Pod itself cycled through
several restarts during this -- expected, not a problem: `Ephemeral
option is enabled` means the runner container exits and re-registers
after every single job, the same self-healing restart pattern already
documented for other repos' runners in this workspace. Confirmed
settled afterward via GitHub's own runner API: `online`, not busy.

[`secrets-manager`](https://github.com/KandlerLi/secrets-manager) itself moved to [`secrets-manager`](https://github.com/KandlerLi/secrets-manager)
2026-09-13, right after the CI automation above landed -- a plain
local directory move (`mv`, git remote unchanged, no GitHub-side
rename) rather than a Terraform/GitHub change, requested directly once
the corrected risk-profile reasoning made it obviously fit `aws/`'s
own grouping criterion (real AWS resources, Terraform-managed,
CI-applied, same as `dyndns`/`website`/`aws-budget`/
`homeserver-health-check`/`ses-relay`) better than `bootstrap/`'s
"rare, admin-privileged, human-applied-only" one it no longer matched.
Every path reference across the workspace was swept and updated to
match -- the repo's own internal self-references (`secrets/*/README.md`,
12 identical occurrences), [`repo-infra/aws_policies.tf`](https://github.com/KandlerLi/repo-infra/blob/main/aws_policies.tf),
[`k3s-apps`](https://github.com/KandlerLi/k3s-apps) (`secrets.tf`, `rotate-sankey-export-app-password.sh`),
[`terraform-state`](https://github.com/KandlerLi/terraform-state) (`operator.tf`, `k3s_bootstrap_local.tf`,
`README.md`), [`dyndns`](https://github.com/KandlerLi/dyndns) (`main.tf`, `outputs.tf`, `README.md`),
`rotate-secrets.md`, the private task list, and the owner's repo map.
Historical narrative in this file is deliberately left as originally
written ("migrated to [`secrets-manager`](https://github.com/KandlerLi/secrets-manager)" stays accurate to
what was true at each of those past moments) rather than rewritten to
match the current location -- this entry is the fix-forward record of
the move itself, not a correction of what came before it.

## GHCR image pulls stay on a classic PAT -- no working alternative exists

`k3s-apps/ghcr-pull-token` (`read:packages`, shared by `modules/
home_agent` and `modules/sankey_export`) stays a classic PAT,
manually rotated, not a GitHub App installation token or a
fine-grained PAT. Both alternatives were seriously investigated
2026-09-17 and are dead ends on GitHub's own platform, not an
engineering-effort problem this workspace could solve by building more
infrastructure:

- **GitHub App installation tokens**: GHCR flatly rejects them for
  `docker pull` regardless of the App's own granted permissions --
  `docker login` succeeds, `docker pull` fails `denied`. Confirmed
  current as of this same month via GitHub's own community discussion
  (github.com/orgs/community/discussions/171423): an official GitHub
  reply acknowledges the gap, with no fix or timeline given a full year
  later. A same-day sibling decision already migrated this workspace's
  *other* durable PAT (the github-runner registration one) to exactly
  this mechanism successfully -- confirming the distinction is real:
  GitHub Apps work fine for runner registration, but GHCR pulls are a
  specific, separate platform gap.
- **Fine-grained PATs**: don't reliably support the `packages: read`
  permission for container registry pulls either -- GitHub's own docs
  and multiple community reports confirm this remains a classic-PAT-only
  feature, removed from GitHub's public roadmap in 2024 with no
  indication it's coming back.

Real, confirmed-live incident that prompted this investigation: the
classic PAT this secret held had already been deleted on GitHub's side
by the time this was checked, silently masked by both images already
being cached on the k3s node (`kubectl describe pod` showed "already
present on machine," not a real pull). A fresh classic PAT
(`read:packages` only) was generated, verified directly against
GitHub's API (`X-OAuth-Scopes: read:packages`) before ever touching AWS,
written into `k3s-apps/ghcr-pull-token` via `put-secret-value`, and
applied (`terraform plan` showed exactly the two expected
`kubernetes_secret_v1.ghcr_pull` resources changing, nothing else).
Revisit this decision only if GitHub ever actually ships GHCR support
for App tokens or fine-grained PAT package permissions -- don't
re-investigate the same dead ends without new evidence that GitHub's
own platform has changed.

## A trivy config scan proves the source is clean, not that a live apply succeeded

[`repo-infra`](https://github.com/KandlerLi/repo-infra)'s `aws_policies.tf` was missing `logs:AssociateKmsKey`/
`logs:DisassociateKmsKey` for `dyndns`'s CI role from the moment
`dyndns#31` first set `kms_key_id` on its own CloudWatch log groups --
every apply after that had actually been failing with
`AccessDeniedException` the whole time. This was tracked at the time as
"applied, confirmed clean," on the strength of a passing `trivy-config`
scan -- but `trivy-config` reads the `.tf` source tree, not live AWS
state, so a scan passing says nothing about whether the plan it
scanned ever actually applied successfully. Found only once the real
apply job's own result was checked directly, not inferred from an
unrelated CI check passing. The standing lesson: a static scan (trivy,
`terraform validate`, `ansible-playbook --syntax-check`) can confirm a
change is well-formed, never that it was actually applied live --
those are two different claims, and only the second one means the
infrastructure is in the state the code describes.

## Two [`gha-common`](https://github.com/KandlerLi/gha-common) CI gotchas, now fixed for every caller

**`aws-actions/configure-aws-credentials` can hang for ~15 minutes on
a self-hosted runner plus container job**, with no real error --
confirmed live, four back-to-back reproductions against `dyndns`
before this workflow was centralized: the step hangs for ~6 minutes on
its first `getIDToken` call, then again for ~8.5 minutes of silence
after credentials actually succeed, before GitHub itself cancels the
whole job. Matches `aws-actions/configure-aws-credentials#1305`, a
self-hosted-runner-plus-container-job OIDC connection issue the
maintainers never root-caused, only added an escape hatch for.
`gha-common`'s own `terraform-checks.yml`/`terraform-apply.yml` set
`action-timeout-s: "90"` on every caller's credentials step -- doesn't
fix the underlying flakiness, but turns a silent ~15-minute hang into
a fast, clear failure.

**A step-level `if:` is required, not a shell-level check inside the
same step, for an optional command input that might be empty.** The
shell-level version (`if [ -n "$CMD" ]; then $CMD; fi`) renders as
`if [ -n "" ]; then<newline><newline>fi` whenever the input is empty
(the common case for every caller except `dyndns`) -- an empty
then-block, which is a POSIX `sh` syntax error ("fi" unexpected"), not
a false condition that just skips the block. Found live failing
`aws-budget`'s very first real Apply run on `main`, right after its
migration PR merged. Fixed by moving the check to the step's own
`if:` field (`if: inputs.extra_validate_command != ''`) instead of
inside the shell script.

## local-exec is allowed only with a live read as its trigger

Decided 2026-09-25. A live check found `pull_request_creation_policy`
set to `collaborators_only` only on `dyndns`; the other 13 `repo-infra`
repositories were on `all`. The self-hosted-runner reasoning above
("dyndns's CI actually runs on the self-hosted runner now") assumes only
the owner can open PRs, so for every repo except `dyndns` that premise
never held. Fork-PR workflow approval was also only
`first_time_contributors` everywhere. No foreign PR or fork workflow run
had happened on any repo.

`terraform-provider-github` still has no resource for either setting.
Julian chose to manage both from `repo-infra` (`pr_policy.tf`) with a
`terraform_data` + `local-exec` running `gh api`, triggered by a
`data "external"` that reads the live value on every plan. This refines
the IaC-only rule rather than breaking it: the rule exists because an
imperative script's drift is invisible to `terraform plan`, and a live
read as the trigger makes drift visible and correctable. A bare
`null_resource`/`local-exec` without such a read stays forbidden: it
runs once and never notices drift, the same failure as the deleted
`protect-repository.sh`. Move to a real provider resource once
upstream issue #3198 lands.

