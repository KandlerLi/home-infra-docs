# ADR 0019: Homeserver monitoring stack and its alerting channel

- Status: Accepted
- Date: 2026-08-24
- Deployment status: applied and live -- `aws/ses-relay` deployed,
  `ansible/roles/monitoring` deployed via `site.yml`, Grafana additionally
  published through `shared_ingress` (see Update below)

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

**Email goes through a new SES relay (`aws/ses-relay`), not the
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
are), and `aws/ses-relay/scripts/derive_smtp_password.py` runs the
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
- `aws/ses-relay` adds one more small AWS-side repository to this
  workspace's now-familiar per-purpose pattern (own Terraform root, own
  `repo-infra` entry, self-hosted CI/CD) — a cost proportional to what it
  actually does (verify one sending domain, one recipient, one scoped
  IAM identity), not a general-purpose mail platform.
- Grafana itself stays internal-only (loopback, no Traefik route) in this
  pass. The same `shared_ingress_<service>_enabled` opt-in pattern every
  other service already uses can be added later if remote dashboard
  access turns out to be wanted — not built now, since it wasn't asked
  for.

## Update (2026-08-25): applied, live bugs fixed, Grafana published

Deploying surfaced several real bugs, none design-level but all worth
recording since the next person touching this code will hit the same
wall otherwise:

- **AWS SES v1 doesn't support resource-level IAM permissions for its
  identity-lifecycle actions.** `ses:VerifyDomainIdentity`,
  `ses:VerifyEmailIdentity`, `ses:VerifyDomainDkim`,
  `ses:GetIdentityVerificationAttributes`, `ses:GetIdentityDkimAttributes`,
  and `ses:DeleteIdentity` were all originally scoped to the two specific
  identity ARNs in `github/repo-infra`'s IAM policy for `ses-relay` —
  each one denied live, one at a time, across three separate applies,
  regardless of the ARN being correct. All six now use `Resource: "*"`;
  SES simply has no finer-grained option for this action group.
- **`for_each` can't iterate a value only known after apply.**
  `aws_route53_record.dkim`'s `for_each` originally iterated
  `aws_ses_domain_dkim.this.dkim_tokens` directly — valid once the
  identity exists, but unusable on the very first plan, since Terraform
  can't determine instance keys from an apply-time value. Fixed by
  iterating static indices (`"0"`, `"1"`, `"2"` — SES always returns
  exactly three DKIM tokens) and indexing into the token list per record.
- **`community.docker.docker_container`'s PID-namespace parameter is
  `pid_mode`, not `pid`** (node_exporter's `pid: host`).
- **A bare `"{{ var }}"` Jinja template preserves the variable's native
  Python type in Ansible, regardless of YAML quoting.**
  `monitoring_grafana_port` is an `int` default; `docker_container`
  rejected it as a non-string env value until `| string` was added.
- **Alertmanager's config file was unreadable to its own container.**
  Installed as `owner: root, mode: 0600`, but the container runs as its
  own dedicated non-root user — fixed to match the Grafana secret file's
  pattern (owned by the container's own user, `0400`).
- **cAdvisor's image-baked healthcheck probes `localhost`, which is
  ambiguous under `network_mode: host`.** The container's `/etc/hosts` is
  the host's own, resolving `localhost` to both `127.0.0.1` and `::1`;
  BusyBox `wget` tried `::1` first, got refused (cAdvisor only binds
  IPv4), and never fell back to IPv4 — reporting unhealthy despite the
  service being fine. Overrode the healthcheck to probe the bind address
  directly, matching Grafana's own healthcheck.
- **blackbox_exporter has no credential to send for Basic-Auth-gated
  targets.** `torrent.jkandler.de` and `ai.jkandler.de` sit behind
  Traefik Basic Auth, and only the bcrypt *hash* is stored anywhere in
  this repo (`shared_ingress_deluge_auth_password_hash` /
  `shared_ingress_agent_auth_password_hash`) — no plaintext credential
  exists for a probe to authenticate with. Probing them with the plain
  `http_2xx` module meant every probe got 401 and `ServiceUnreachable`
  fired permanently. Split into a second probe group
  (`monitoring_probe_targets_authenticated`) using a new
  `http_2xx_or_401` blackbox module: 401 still proves DNS/TLS/Traefik/
  the upstream are all up and correctly gating access, so real outages
  (timeouts, 5xx, connection refused) still alert — it just can't
  confirm the app behind auth is itself healthy.

**Grafana is now published through `shared_ingress`** at
`grafana.jkandler.de`, reversing the "stays internal-only" call above —
the user asked for it directly once the stack was live and useful enough
to want checking from outside an SSH tunnel. Same defense-in-depth shape
as Deluge: its own subdomain, its own Traefik-level Basic Auth credential
(independent of every other route's), matching rate limits/security
headers, and the same one-time `PUBLISH_GRAFANA` confirmation gate plus
health check the other routes use — Grafana's own admin login stays in
place underneath, layered behind the Traefik gate rather than replaced
by it. `dyndns` gained a `grafana` CNAME to support it.

**The ntfy raw-JSON limitation (see Consequences above) was confirmed
live and deliberately left as-is.** Asked directly whether to build a
relay to reformat it: the user chose to park it — email already carries
the full readable detail, so ntfy stays the "something's wrong, check
your email" buzz, and building a translator for a purely cosmetic gap
wasn't judged worth the new container/maintenance surface.

## Update (2026-08-25): SES email delivery — two more real bugs, now confirmed live

The dual-channel test alert in the previous Update section confirmed
ntfy delivery but not email — nobody had actually checked the inbox yet.
Following up surfaced two further, genuine bugs, both now fixed and both
confirmed live:

- **A truncated SMTP username.** `monitoring_ses_smtp_username` in SOPS
  held `AKIA4NU5Y52WI2R4RTE` — 19 characters, missing the trailing `4`
  from the real 20-character access key ID (`...RTE4`). A copy/paste
  slip, invisible until Alertmanager's logs showed
  `535 "Authentication Credentials Invalid"` on every send attempt. A
  first attempted fix didn't actually take (the templated
  `alertmanager.yml` came back `ok`, not `changed`, on the next
  `site.yml` run — a tell that the underlying value hadn't changed);
  `sops set` (rather than an interactive editor) was used the second
  time and confirmed via the task showing `changed`.
- **SES send authorization scoped to the wrong identity.** With the
  credentials fixed, sending then failed with
  `554 "Access denied: ... not authorized to perform 'ses:SendRawEmail'
  on resource 'arn:aws:ses:...:identity/julian.kandler@outlook.com'"` —
  the *recipient's* identity ARN, not the sender's. `ses-relay`'s IAM
  policy for the SMTP user had only ever scoped `ses:SendRawEmail`/
  `ses:SendEmail` to the sending domain identity
  (`aws_ses_domain_identity.this.arn`). SES authorizes a send against
  every identity involved, not just the source, when the recipient is
  also a verified identity in the same account (a sandbox requirement
  here, since the account never requested production access) — the
  policy needed `aws_ses_email_identity.recipient.arn` added to its
  `Resource` list too (`ses-relay` PR #2).

With both fixed, a further test alert produced no error at all in
Alertmanager's logs, and the user confirmed the email actually arrived.
Both alerting channels are now independently verified end to end, not
just "Alertmanager attempted delivery."

## Update (2026-08-28 / 2026-09-07): three more real bugs found live

- **`ansible.builtin.getent` does not accumulate looped keys into
  `ansible_facts.getent_passwd`.** Each loop iteration *replaces* that
  fact with a dict containing only the one key just looked up, so
  looping this task over several accounts leaves only the last
  iteration's key by the time the loop finishes (reproduced directly: a
  two-item loop over `root`/`daemon` left only `daemon`). Account
  creation itself loops fine (`ansible.builtin.user` has no such shared
  state to clobber); the `getent` + `set_fact` identity lookup stays as
  one task pair per account instead.
- **A shared parent directory takes its owner/mode from whichever task
  creates it first, not from a later task that also touches it.**
  `ansible.builtin.file` on a path whose parent doesn't exist yet
  creates that parent stamped with the *current* task's own
  owner/mode — so a loop installing several sibling directories under
  one shared root (e.g. `monitoring_data_dir/prometheus`,
  `.../ntfy-relay`) can leave the shared root itself owned by whichever
  one happened to run first, at a restrictive mode, silently blocking
  every other account from even traversing into it. Never surfaced for
  the Docker-based services here (a bind mount doesn't traverse the
  host's real parent-directory chain the way a native process does) —
  found only once `ntfy-relay.service`, the first native (non-container)
  reader under that tree, got `EACCES` opening its own correctly-owned
  file. Fixed with a dedicated task creating the shared parent first,
  ahead of the loop.
- **`community.docker.docker_image` reports `changed` on every run when
  pulling a combined `name:tag@digest` reference.** Docker itself
  leaves the local image stored with an untagged (`<none>`) tag when
  pulled that way (confirmed via `docker images -a`), even though the
  running container uses that exact image correctly. The module's own
  idempotency check looks for a local image matching the full pull
  string, never finds one, and pulls "again" every time —  a real
  Docker behavior, not a module bug. Pulling by digest alone (dropping
  the tag from the standalone pull task, while leaving the
  `docker_container` tasks' own `name:tag@digest` references untouched,
  since their own image comparison already worked correctly) sidesteps
  it.
