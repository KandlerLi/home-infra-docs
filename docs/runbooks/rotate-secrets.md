# Runbook: Rotate and Manage AWS Secrets Manager Secrets

Status: current. Covers all 10 secret groups / 26 keys created by the
SOPS-to-Secrets-Manager cutover (`decisions.md`'s own
writeups). Every `aws_secretsmanager_secret` container resource has
since migrated out of [`terraform-state`](https://github.com/KandlerLi/terraform-state) (that root's own
`secrets_manager.tf` is deleted, the campaign closed out 2026-09-12)
into `[`secrets-manager`](https://github.com/KandlerLi/secrets-manager)/` (a self-contained module dir per
secret, `secrets/<repo>/<service>/`) — see that repo's own README for
the migration-status table; each secret's own `README.md` there
documents its keys, consumers, and rotation procedure. [`home-infra`](https://github.com/KandlerLi/home-infra)'s own README.md ("First apply" section)
covers a plain *edit* -- this covers what rotating each kind of value
actually involves, and whether a given secret still belongs in Secrets
Manager at all. (`security-and-secrets.md` is not a reliable companion
here -- confirmed 2026-09-10 that its own "Secret management" section
still describes SOPS and pre-consolidation `shared_ingress_agent_auth_*`/
`shared_ingress_deluge_auth_*` variable names, both gone; see
the private task list.)

## The generic edit, for reference

Every rotation below is a variation on this: read the group's current
JSON blob, merge in the new value for one key, write the whole blob
back (never overwrite a group with a single key's value -- each group
holds several unrelated keys):

```bash
aws secretsmanager get-secret-value --secret-id <group> --query SecretString --output text \
  | jq '.<key> = "<new-value>"' > /tmp/secret.json
aws secretsmanager put-secret-value --secret-id <group> --secret-string file:///tmp/secret.json
rm /tmp/secret.json
```

After writing, the consuming Terraform root needs a real
`plan`/`apply` (`jsondecode()` reads the live value, so nothing
rebuilds automatically) and/or the Ansible playbook needs a real run,
depending on which side actually consumes the key -- see each group's
own file under [`secrets-manager/secrets`](https://github.com/KandlerLi/secrets-manager/blob/main/secrets) for who that is, or
`repositories.md`'s repo-by-repo breakdown.

## Category A: plain credentials -- rotate anytime, no coordination needed

`home_agent_anthropic_api_key` and `home_agent_openai_api_key` (both
`home-infra/home-agent`) and `token` (`k3s-apps/ghcr-pull-token` —
split out of `home-infra/home-agent` 2026-09-12, shared by
`modules/home_agent` and `modules/sankey_export`). Generate the new
credential at its own source (Anthropic Console, OpenAI dashboard, a
new GHCR PAT), `put-secret-value`, then `terraform apply` in
[`k3s-apps`](https://github.com/KandlerLi/k3s-apps) to roll the consuming Deployment/Secret.
`home_agent_anthropic_api_key` is home_agent's own chat/tool-use
credential (`agent.py`'s `AnthropicMessagesProvider`);
`home_agent_openai_api_key` moved to Whisper speech-to-text only
(`audio.py`) once chat switched providers 2026-09-18 — both are live
and independently rotatable.

`k3s_ingress_acme_dns01_access_key_id` / `_secret_access_key`
(`home-infra/ingress`) is the same shape but its own credential is
itself Terraform-managed (`aws_iam_access_key.acme_dns01` in
[`dyndns`](https://github.com/KandlerLi/dyndns), `lifecycle { create_before_destroy = true }`) -- rotate
it with `terraform apply -replace=aws_iam_access_key.acme_dns01` there
instead of a raw `aws iam create-access-key` (which would create a
real IAM key Terraform's own state doesn't know about). See
[`secrets-manager/secrets/home-infra/ingress/README.md`](https://github.com/KandlerLi/secrets-manager/blob/main/secrets/home-infra/ingress/README.md) for
the full proven procedure (rotated live 2026-09-11) and why the old
key needs no separate manual deletion step any more.

`blocky_postgres_password` is almost this category, but the database
itself also needs the change -- Postgres won't accept the new value
just because Secrets Manager has it:

```bash
kubectl -n apps exec -it deploy/blocky -c postgres -- psql -U blocky -c \
  "ALTER USER blocky WITH PASSWORD '<new-password>';"
```

then `put-secret-value` and `terraform apply` (which updates the
Kubernetes Secret Blocky itself reads) -- in that order, so the old
value still works for the brief gap between the `ALTER USER` and the
apply picking it up, rather than the other way around locking Blocky
out.

## Category B: Authelia's own internal crypto material -- rare, real blast radius

All in `home-infra/authelia`. Not something to rotate casually --
each of these has a real, different cost:

- **`authelia_session_secret`** -- rotating logs out every active
  session immediately (everyone re-authenticates, MFA included).
  Otherwise safe; generate a new one with `authelia crypto rand
  --length 64 --charset alphanumeric` (same tool Authelia itself
  ships, `ghcr.io/authelia/authelia:4.39.22` per
  [`k3s-apps/modules/authelia/main.tf`](https://github.com/KandlerLi/k3s-apps/blob/main/modules/authelia/main.tf)'s pinned image).
- **`authelia_storage_encryption_key`** -- encrypts session/TOTP state
  inside Authelia's own SQLite file. **Do not just swap the value** --
  that leaves the database encrypted with the old key while Authelia
  expects the new one, breaking every existing TOTP registration.
  Authelia ships a dedicated command for this exact case:
  `authelia storage encryption change-key`, run against the live
  database, with the Secrets Manager value updated to match before the
  Pod restarts. The script below writes Secrets Manager first and then
  runs `change-key`, so an interrupted run can never lose the new key.
- **`authelia_reset_password_jwt_secret`** -- low-stakes; only
  invalidates any password-reset email link currently in flight (rare
  in practice). Generate the same way as the session secret.
- **`authelia_oidc_hmac_secret`** and **`authelia_oidc_issuer_private_key`**
  -- rotating either one invalidates every live OIDC session across
  Grafana, Open WebUI, and Nextcloud simultaneously (all three
  services' users get logged out and must re-authenticate through
  Authelia). Generate with `authelia crypto rand --length 64 --charset
  alphanumeric` (hmac secret) or `authelia crypto pair rsa generate -b
  4096` (issuer private key, PEM/4096-bit).

For all five: `put-secret-value` on `home-infra/authelia`, then
`terraform apply` in [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) to roll Authelia's own
Deployment. Plan for the login/re-auth disruption -- there's no way to
rotate these without one.

The session secret, storage key and issuer key go together through
[`k3s-apps/scripts/rotate-authelia-internal-secrets.sh`](https://github.com/KandlerLi/k3s-apps/blob/main/scripts/rotate-authelia-internal-secrets.sh)
(first used 2026-10-25): backup job, values generated in memory,
Secrets Manager, `change-key`, check, `terraform apply`. It needs
`aws login` as an identity that can also read `k3s-apps`' state, and
`kubectl -n apps port-forward svc/stalwart-internal 18083:8080` for the local
apply. Details and the recovery state when it stops after `change-key`:
[`secrets-manager`](https://github.com/KandlerLi/secrets-manager)'s `runbooks/home-infra/authelia.md`.

## Category C: Authelia OIDC client secret pairs -- coordinated, two secrets + two redeploys

One pair per client (Grafana/Open WebUI/Nextcloud): Authelia holds a
hash (`home-infra/authelia`'s `authelia_oidc_*_client_secret_hash`),
the client itself holds the plaintext (Grafana in
`home-infra/grafana`, Open WebUI in `home-infra/open-webui`, Nextcloud
in `home-infra/nextcloud` -- Nextcloud's plaintext flows through
[`home-infra`](https://github.com/KandlerLi/home-infra)'s own Ansible, not Terraform, since Nextcloud AIO
runs on the homeserver, not in k3s). Rotating one without the other
breaks that client's SSO outright -- the guide is the exact sequence,
not just "change both":

1. Generate a fresh plaintext + its hash together, in one command
   ([`k3s-apps/modules/authelia/variables.tf`](https://github.com/KandlerLi/k3s-apps/blob/main/modules/authelia/variables.tf)'s own documented
   generation for each pair):
   ```bash
   docker run --rm authelia/authelia:4.39.22 \
     authelia crypto hash generate pbkdf2 --variant sha512 \
     --random --random.length 64 --random.charset alphanumeric
   ```
   This prints both the random plaintext it generated and its pbkdf2
   hash -- keep both, you need them for two different secrets.
2. Feed both values to
   `[`k3s-apps/scripts/rotate-oidc-client-secret.sh`](https://github.com/KandlerLi/k3s-apps/blob/main/scripts/rotate-oidc-client-secret.sh) grafana` (or
   `openwebui`, or `nextcloud`) -- it prompts silently for the
   plaintext and the hash, rejects an obviously-swapped pair (a pbkdf2
   hash always starts with `$`, the random plaintext never does),
   writes both secrets, and runs `terraform apply` (rolls Authelia's
   own side, plus the client's Deployment for Grafana/Open WebUI --
   Nextcloud has none, it runs on the homeserver). **Nextcloud** (script
   support added 2026-09-11) additionally re-runs [`home-infra`](https://github.com/KandlerLi/home-infra)'s
   `site.yml` itself (`--ask-become-pass`, run this one yourself) so
   the new plaintext reaches Nextcloud's own `occ user_oidc:provider`
   config -- confirmed live this step is a real upsert, not a
   formality.

Getting the order wrong (only updating one side, or applying the two
repos' changes far apart in time) locks that one client's SSO out
until both sides match again -- native login stays disabled for
Grafana/Open WebUI regardless (see Category I), so a mismatch there is
a real, if temporary, outage for that service.

## Category D: personal login credentials (Julian's own)

- **`authelia_admin_password_hash`** (`home-infra/authelia`) -- your
  real Authelia login. Generate a new hash with `docker run --rm
  authelia/authelia:4.39.22 authelia crypto hash generate argon2
  --password '<new-password>'`, `put-secret-value`, `terraform apply`.
  Update your own password manager entry *before* applying, not after
  -- there's no way to look the plaintext back up once only the hash
  exists.
- ~~`shared_ingress_auth_password` + `shared_ingress_auth_password_hash`~~
  -- **retired 2026-09-11**, not just this entry gone stale. This was
  Traefik's dormant Basic Auth fallback; once `authelia-forward-auth`
  held stable across several more days with no recurrence of
  the private task list's Authelia SQLite issue, the whole rollback path (the
  `shared-auth` middleware, its Kubernetes Secret, this key pair, and
  `sync_secrets_to_pass.py` itself, whose only remaining mapping this
  was) was removed rather than rotated. See
  `home-infra/ingress/README.md`'s own "Removed" section.

## Category E: Nextcloud app passwords (dedicated bot accounts)

`nextcloud_tools_app_password` (`home-infra/home-agent`),
`sankey_export_app_password` (`k3s-apps/sankey-export`).

**Hard wall on automation:** a Nextcloud app password cannot mint
another one, so minting the replacement always needs `occ` on the
homeserver (or the account's login password, which is generated and
discarded at bootstrap per ADR 0008). Nothing reachable from the
in-cluster runner or an AWS Lambda can do it. Everything *after* the
mint is scriptable.

```bash
# on the homeserver -- the one manual step
docker exec nextcloud-aio-nextcloud php occ user:auth-tokens:add \
  --name "rotate-$(date +%F)" -- <nextcloud_tools|sankey-export>
```

For **`sankey_export_app_password`**, feed that token to
[`k3s-apps/scripts/rotate-sankey-export-app-password.sh`](https://github.com/KandlerLi/k3s-apps/blob/main/scripts/rotate-sankey-export-app-password.sh) -- it does
the `put-secret-value`, the `terraform apply`, waits for the next
CronJob run to prove the token, and prints the revoke command.

For **`nextcloud_tools_app_password`** there is no helper script yet
([`home-infra`](https://github.com/KandlerLi/home-infra)'s own `nextcloud_tools` role and its rotation
playbook were retired when `home_agent`/`open_webui` moved to k3s-native,
`018c5ff`): capture the token, `put-secret-value` it into
`home-infra/home-agent`, `terraform apply` in [`k3s-apps`](https://github.com/KandlerLi/k3s-apps).

Either way, revoke the superseded token only after confirming the new
one works: `occ user:auth-tokens:list <account>` to get the ids, then
`occ user:auth-tokens:delete <account> <id>` (note the arg order --
`:list`/`:delete` take the account positionally *first*, `:add` takes it
*last*). Both tokens are valid simultaneously until you revoke one, so
there's no blackout window if you check first.

## Category F: `github_runner_github_token`

Fine-grained GitHub PAT with `Administration: write` on every
self-hosted-runner repository. Generate a new one in GitHub's own PAT
settings (same scope as the current one), `put-secret-value` on
`home-infra/github-runner`, then an actual [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap)
apply. (Retired: since 2026-09-16 the runners use the GitHub App
instead, since 2026-11-01 through ARC's `arc-github-app` Secret.)
Revoke the old PAT in GitHub only after confirming at least one
runner re-registered successfully post-apply.

## Category G: `monitoring_ses_smtp_password` -- derived, not chosen

Not a password you invent -- it's derived from a real SES secret
access key via [`ses-relay/scripts/derive_smtp_password.py`](https://github.com/KandlerLi/ses-relay/blob/main/scripts/derive_smtp_password.py), AWS's
own published SigV4-based algorithm. The underlying IAM access key is
itself Terraform-managed (`aws_iam_access_key.smtp` in [`ses-relay`](https://github.com/KandlerLi/ses-relay),
`lifecycle { create_before_destroy = true }`, same pattern as
`home-infra/ingress`'s ACME key) -- rotate it with `terraform apply
-replace=aws_iam_access_key.smtp` there, not a raw `aws iam
create-access-key`. Rotated live 2026-09-11: derive script against the
new key, `put-secret-value` on `home-infra/monitoring`, `terraform
apply` in [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) -- rolls *both* Alertmanager and Authelia
automatically (`checksum/config` on each Deployment). No `site.yml`
re-run needed -- confirmed live that Ansible's own `pre_tasks` only
`assert` these values are non-empty, never template them into a file
(Alertmanager itself moved to k3s outright). Verified for real with a
genuine SMTP AUTH + send against SES using the new credentials, not
just a clean apply. See
[`secrets-manager/secrets/home-infra/monitoring/README.md`](https://github.com/KandlerLi/secrets-manager/blob/main/secrets/home-infra/monitoring/README.md)
for the full procedure.

## Category H: `monitoring_ntfy_topic` -- an identifier, not a credential

Not a password in the usual sense -- ntfy's own security model treats
an unguessable topic *name* as the actual secret (anyone who knows it
can publish to it), so it still belongs in Secrets Manager, but
"rotating" it just means picking a new unguessable string and updating
both the publisher (wherever alerts get sent from) and subscriber
(the ntfy app/client) to the new topic name -- there's no cryptographic
material to regenerate.

## Category I: removed -- `monitoring_grafana_admin_password` (2026-09-10)

Grafana's native login form *and* HTTP Basic Auth are both disabled at
the protocol level (`GF_AUTH_DISABLE_LOGIN_FORM` +
`GF_AUTH_BASIC_ENABLED=false` in [`k3s-apps/modules/grafana`](https://github.com/KandlerLi/k3s-apps/blob/main/modules/grafana)) --
Authelia OIDC is the only way in -- so this value could not be rotated
into a *working* credential; there was no login path left that accepts
it. It was **removed** from the `home-infra/grafana` group's JSON:
`modules/grafana` now fills `GF_SECURITY_ADMIN_PASSWORD__FILE` from a
module-local `random_password` (random, not a fixed placeholder, so an
accidental rollback of those two `GF_AUTH_*` switches doesn't hand out a
known-value admin login), and the `grafana/user`/`grafana/password`
`pass` entries + the `monitoring_grafana_admin_user` Ansible default
went with it. Nothing to rotate here.

## Rotation reminders -- shipped, not the same thing as automated rotation

[`secrets-manager`](https://github.com/KandlerLi/secrets-manager)'s own `rotation-schedule.json` +
`.github/workflows/check-secret-rotation.yml` (2026-09-17) email a
warning once a listed secret is within 7 days of a manually-recorded
deadline -- a **reminder**, distinct from the "actually rotate it
unattended" problem the rest of this section covers below. Rotation
itself still stays manual; this only makes staleness visible instead
of silent. Only lists secrets with a genuine calendar-driven deadline
(right now: `k3s-apps/ghcr-pull-token`'s own GitHub-enforced 90-day
classic-PAT expiry) -- see that repo's own README.md for the full
mechanism and how to add another secret to the list.

## Automated rotation -- reality check before building it

AWS Secrets Manager's own native rotation is Lambda-based, with
out-of-box templates for RDS/DocumentDB/Redshift-style databases.
**Nothing in this workspace is an AWS-managed database** -- even
`blocky_postgres_password` is a self-hosted, in-cluster Postgres, not
RDS, so the native templates don't apply to a single secret here.
Any automated rotation needs a custom Lambda implementing the
four-step create/set/test/finish protocol, reaching into the k3s
cluster (or GitHub's API, or Nextcloud's `occ`, or Authelia's own CLI)
to actually change the downstream credential -- a real, nontrivial
build, not a checkbox AWS provides.

Realistic prioritization if this gets picked up:

- **Worth automating**: Category A's plain credentials and
  `blocky_postgres_password` -- mechanically simple, low blast radius,
  the kind of thing a Lambda with cluster access can safely do
  unattended.
- **Probably not worth automating**: Category B (Authelia's crypto
  material) -- the blast radius (logging everyone out, invalidating
  live OIDC sessions) makes *scheduled, unattended* rotation actively
  harmful, not just unnecessary; this should stay a deliberate,
  human-triggered action regardless of whether the mechanics get
  scripted.
- **Structurally resistant to automation**: Category C (OIDC pairs --
  two secrets plus two coordinated redeploys, in two different repos
  for the Nextcloud case) and Category D (personal credentials -- a
  human has to actually know the new plaintext to keep logging in).

## Storage-location flags

Everything else in Secrets Manager needs to stay there -- Terraform's
own `jsondecode()` ([`k3s-apps/secrets.tf`](https://github.com/KandlerLi/k3s-apps/blob/main/secrets.tf),
[`k3s-bootstrap/secrets.tf`](https://github.com/KandlerLi/k3s-bootstrap/blob/main/secrets.tf)) reads the real value at every
`plan`/`apply`, and Ansible's `pre_tasks`
([`home-infra/ansible/playbooks/site.yml`](https://github.com/KandlerLi/home-infra/blob/main/ansible/playbooks/site.yml)) reads it at every
`site.yml` run -- moving any of those out to `pass`-only storage would
reintroduce exactly the manual-export step the SOPS-to-Secrets-Manager
cutover was built to remove. The real exceptions found so far:

- **`monitoring_grafana_admin_password`** -- **removed** 2026-09-10
  (Category I above). Not moved to local storage: deleted outright.
- **`shared_ingress_auth_password`/`_hash`** -- stays in Secrets
  Manager for now (Terraform actively keeps the Kubernetes Secret
  populated so the rollback stays a one-chain-edit swap, no rebuild),
  but it's a real trade-off worth revisiting if the underlying
  Authelia storage issue goes long enough without
  recurring that the rollback path stops earning its keep.

No other secret is a plausible candidate for local-only storage --
every one either feeds a live Terraform/Ansible read, or (Category
D's other entry, `authelia_admin_password_hash`) genuinely needs to be
wherever the service that checks it against Julian's real login can
reach at deploy time.
