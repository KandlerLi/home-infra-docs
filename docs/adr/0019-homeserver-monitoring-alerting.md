# ADR 0019: Homeserver monitoring stack and its alerting channel

- Status: Accepted
- Date: 2026-08-24
- Deployment status: `ansible/roles/monitoring` and `infra/ses-relay` built
  and locally validated, not yet applied

## Context

ADR 0017 scoped the homeserver-hosted half of monitoring (host health,
container health, service reachability, certificate/DNS expiry) as
Prometheus + Grafana, explicitly deferred as "a separate, larger
follow-up." This ADR covers building that follow-up, and the one thing
ADR 0017 left genuinely open: how it actually notifies anyone.

## Decisions

**Prometheus, Grafana, node_exporter, cAdvisor, blackbox_exporter, and
Alertmanager** — six containers, one role (`ansible/roles/monitoring`),
matching `nextcloud_aio`'s "one role, many containers" shape rather than
one role per container. blackbox_exporter's HTTPS probe module covers
both service reachability *and* certificate expiry in a single tool
(`probe_ssl_earliest_cert_expiry` comes free with every HTTPS probe), so
no separate certificate-expiry exporter was needed.

**Every container runs on `network_mode: host`**, not a mix of host and
bridge networking. This was a real bug caught before deployment: with
some containers on Docker's bridge network and `published_ports`,
`127.0.0.1` inside those containers resolves to their *own* loopback, not
the host's — Prometheus would never have actually reached node_exporter,
or anything else, despite every scrape config looking correct. Putting
everything on host networking (the same precedent `shared_ingress`
already established) means every scrape target is uniformly
`127.0.0.1:<port>`, with no Docker network translation to get subtly
wrong.

**Alertmanager, not Grafana's built-in unified alerting.** The original
plan avoided a separate Alertmanager container specifically to keep the
stack one container leaner. Reconsidered while actually writing the
config: Grafana's alerting provisioning schema requires each rule's
query and threshold expressed as a multi-step pipeline (a data source
query plus a separate `__expr__` reduce/threshold step), which is real,
concrete risk to get exactly right without a live Grafana instance to
verify against — unlike Prometheus's own native alerting rule format
(plain PromQL, `alert`/`expr`/`for`), which is simple, extremely
well-documented, and already correctly written. Alertmanager itself is a
single small Go binary — the resource cost of a sixth container is not
actually worth avoiding for the correctness risk it would have traded
away.

**Alerting is dual-channel: ntfy push notification and email, both wired
to every alert.** Asked directly, since ADR 0017 left this open — the
user wants both, not a choice between them, specifically so one channel
being misconfigured doesn't mean total silence.

**Email goes through a new SES relay (`infra/ses-relay`), not the
existing Outlook address.** The original assumption — an SMTP app
password for `julian.kandler@outlook.com`, the same channel already used
for AWS Budget/health-check alerts — turned out not to work: as of 2026,
Microsoft has retired app-password/basic-auth SMTP entirely for personal
Outlook.com accounts, leaving only OAuth2, which Alertmanager (like
virtually every self-hosted alerting tool) can't do. SES can still email
*to* that same Outlook address as the recipient — it just can't relay
*through* Outlook's own SMTP anymore. `ses-relay` is its own repository
rather than folded into `homeserver-health-check`, whose own README
already states it's deliberately kept small and not meant to grow into a
general-purpose AWS monitoring stack.

**The SES SMTP password is derived outside Terraform, on purpose.** SES's
SMTP password isn't a value AWS stores anywhere — it's a documented,
deterministic transform of the IAM secret access key (the same algorithm
the SES console's own credential wizard runs). Computing it inside
Terraform would put the actual, usable SMTP password in state, on top of
the raw IAM secret key already there. Instead, Terraform outputs the raw
secret key (marked sensitive, as `aws_iam_access_key` outputs normally
are), and `infra/ses-relay/scripts/derive_smtp_password.py` runs the
transform locally, once, with the result going straight into SOPS —
never through Terraform state, matching this workspace's existing pattern
for other one-time, can't-automate secrets (the GPG passphrase, the SNS/
SES email confirmations).

**Grafana's admin password uses the `_FILE` env-var convention, not a raw
env var.** Matches `home_agent`'s own `OPENAI_API_KEY_FILE` pattern —
neither is visible via `docker inspect`, unlike a value passed through
plain `env:`.

**Prometheus's data lives on the root disk, not the HDD**, with retention
capped by both time (30 days) and size (2 GB): the root disk had only
23 GiB free at last check, and the HDD is spinning disk already busy
with Nextcloud/Deluge I/O — a small, bounded TSDB is a better fit for
either constraint than either extreme.

## Consequences

- The ntfy push channel shows Alertmanager's raw JSON payload as the
  notification text rather than a formatted summary — ntfy doesn't
  natively understand Alertmanager's webhook payload shape. A
  notification does arrive; making it readable is a deliberately
  deferred follow-up, not solved here.
- `infra/ses-relay` adds one more small AWS-side repository to this
  workspace's now-familiar per-purpose pattern (own Terraform root, own
  `repo-infra` entry, self-hosted CI/CD) — a cost proportional to what it
  actually does (verify one sending domain, one recipient, one scoped
  IAM identity), not a general-purpose mail platform.
- Grafana itself stays internal-only (loopback, no Traefik route) in this
  pass. The same `shared_ingress_<service>_enabled` opt-in pattern every
  other service already uses can be added later if remote dashboard
  access turns out to be wanted — not built now, since it wasn't asked
  for.
