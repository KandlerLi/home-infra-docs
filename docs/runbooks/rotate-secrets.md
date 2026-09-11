# Runbook: Rotate and Manage AWS Secrets Manager Secrets

Status: current. Covers all 10 secret groups / 26 keys created by the
SOPS-to-Secrets-Manager cutover (`PARKED.md`/`decisions.md`'s own
writeups). The `aws_secretsmanager_secret` container resources are being
migrated out of `bootstrap/terraform-state/secrets_manager.tf` into
`bootstrap/secrets-manager/` one group at a time (a self-contained
module dir per secret, `secrets/<repo>/<service>/`) — see that repo's
own README for the migration-status table; each secret's own
`README.md` there names where its resource currently lives. `infra/home-infra`'s own README.md ("First apply" section)
covers a plain *edit* -- this covers what rotating each kind of value
actually involves, and whether a given secret still belongs in Secrets
Manager at all. (`security-and-secrets.md` is not a reliable companion
here -- confirmed 2026-09-10 that its own "Secret management" section
still describes SOPS and pre-consolidation `shared_ingress_agent_auth_*`/
`shared_ingress_deluge_auth_*` variable names, both gone; see
`BACKLOG.md`.)

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
own file under `bootstrap/secrets-manager/secrets/` for who that is, or
`repositories.md`'s repo-by-repo breakdown.

## Category A: plain credentials -- rotate anytime, no coordination needed

`home_agent_openai_api_key`, `home_agent_ghcr_token`
(`home-infra/home-agent`), `k3s_ingress_acme_dns01_access_key_id` /
`_secret_access_key` (`home-infra/ingress`). Generate the new
credential at its own source (OpenAI dashboard, a new GHCR token,
`aws iam create-access-key` on `dyndns`'s own `traefik-acme-dns01` IAM
user), `put-secret-value`, then `terraform apply` in `infra/k3s-apps`
to roll the consuming Deployment/Secret. For the ACME DNS-01 pair,
delete the old access key from IAM only after confirming the new one
issued a real certificate -- Let's Encrypt renewal runs continuously,
so there's no safe window to be credential-less.

`blocky_postgres_password` is almost this category, but the database
itself also needs the change -- Postgres won't accept the new value
just because Secrets Manager has it:

```bash
kubectl exec -it deploy/blocky -c postgres -- psql -U blocky -c \
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
  `infra/k3s-apps/modules/authelia/main.tf`'s pinned image).
- **`authelia_storage_encryption_key`** -- encrypts session/TOTP state
  inside Authelia's own SQLite file. **Do not just swap the value** --
  that leaves the database encrypted with the old key while Authelia
  expects the new one, breaking every existing TOTP registration.
  Authelia ships a dedicated command for this exact case:
  `authelia storage encryption change-key`, run against the live
  database before rotating the Secrets Manager value to match.
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
`terraform apply` in `infra/k3s-apps` to roll Authelia's own
Deployment. Plan for the login/re-auth disruption -- there's no way to
rotate these without one.

## Category C: Authelia OIDC client secret pairs -- coordinated, two secrets + two redeploys

One pair per client (Grafana/Open WebUI/Nextcloud): Authelia holds a
hash (`home-infra/authelia`'s `authelia_oidc_*_client_secret_hash`),
the client itself holds the plaintext (Grafana in
`home-infra/grafana`, Open WebUI in `home-infra/open-webui`, Nextcloud
in `home-infra/nextcloud` -- Nextcloud's plaintext flows through
`infra/home-infra`'s own Ansible, not Terraform, since Nextcloud AIO
runs on the homeserver, not in k3s). Rotating one without the other
breaks that client's SSO outright -- the guide is the exact sequence,
not just "change both":

1. Generate a fresh plaintext + its hash together, in one command
   (`infra/k3s-apps/modules/authelia/variables.tf`'s own documented
   generation for each pair):
   ```bash
   docker run --rm authelia/authelia:4.39.22 \
     authelia crypto hash generate pbkdf2 --variant sha512 \
     --random --random.length 64 --random.charset alphanumeric
   ```
   This prints both the random plaintext it generated and its pbkdf2
   hash -- keep both, you need them for two different secrets.
2. **Grafana or Open WebUI**: feed both values to
   `infra/k3s-apps/scripts/rotate-oidc-client-secret.sh grafana` (or
   `openwebui`) -- it prompts silently for the plaintext and the hash,
   rejects an obviously-swapped pair (a pbkdf2 hash always starts with
   `$`, the random plaintext never does), writes both secrets, and runs
   `terraform apply` (rolls Authelia and the client Deployment together,
   in sync). **Nextcloud** has no helper script yet: `put-secret-value`
   the hash into `home-infra/authelia`, the plaintext into
   `authelia_oidc_nextcloud_client_secret` in `home-infra/nextcloud`,
   `terraform apply` in `infra/k3s-apps` for Authelia, then re-run
   `infra/home-infra`'s `site.yml` so the new plaintext reaches
   `nextcloud_aio`'s own OIDC config.

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
- **`shared_ingress_auth_password`** + **`shared_ingress_auth_password_hash`**
  (`home-infra/ingress`) -- Traefik's dormant Basic Auth fallback (see
  `BACKLOG.md`'s Authelia SQLite item for why it's kept). Generate a
  new password, hash it the same way as `authelia_admin_password_hash`
  above (any bcrypt tool works here, not just Authelia's -- Traefik's
  `basicAuth` middleware just wants a standard htpasswd-format hash),
  `put-secret-value` both keys, `terraform apply`, and update
  `infra/home-infra/scripts/sync_secrets_to_pass.py`'s target
  (`ingress/password` in `pass`) by re-running that script so your own
  `pass` copy doesn't go stale.

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
`infra/k3s-apps/scripts/rotate-sankey-export-app-password.sh` -- it does
the `put-secret-value`, the `terraform apply`, waits for the next
CronJob run to prove the token, and prints the revoke command.

For **`nextcloud_tools_app_password`** there is no helper script yet
(`infra/home-infra`'s own `nextcloud_tools` role and its rotation
playbook were retired when `home_agent`/`open_webui` moved to k3s-native,
`018c5ff`): capture the token, `put-secret-value` it into
`home-infra/home-agent`, `terraform apply` in `infra/k3s-apps`.

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
`home-infra/github-runner`, then an actual `bootstrap/k3s-bootstrap`
apply -- all 7 repos' runner Deployments share one Kubernetes Secret
(`modules/github_runner`), so one apply rolls every runner at once.
Revoke the old PAT in GitHub only after confirming at least one
runner re-registered successfully post-apply.

## Category G: `monitoring_ses_smtp_password` -- derived, not chosen

Not a password you invent -- it's derived from a real SES secret
access key via `aws/ses-relay/scripts/derive_smtp_password.py`, AWS's
own published SigV4-based algorithm. "Rotating" this really means
rotating the underlying IAM access key first (`aws/ses-relay`'s own
Terraform-managed SES user), running the derive script against the
new key, then `put-secret-value` on `home-infra/monitoring` and
`terraform apply` in `infra/k3s-apps` (Alertmanager) --
`infra/home-infra`'s own `site.yml` pre_tasks also need a re-run since
Ansible reads this same group directly.

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
`GF_AUTH_BASIC_ENABLED=false` in `infra/k3s-apps/modules/grafana`) --
Authelia OIDC is the only way in -- so this value could not be rotated
into a *working* credential; there was no login path left that accepts
it. It was **removed** from the `home-infra/grafana` group's JSON:
`modules/grafana` now fills `GF_SECURITY_ADMIN_PASSWORD__FILE` from a
module-local `random_password` (random, not a fixed placeholder, so an
accidental rollback of those two `GF_AUTH_*` switches doesn't hand out a
known-value admin login), and the `grafana/user`/`grafana/password`
`pass` entries + the `monitoring_grafana_admin_user` Ansible default
went with it. Nothing to rotate here.

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
own `jsondecode()` (`infra/k3s-apps/secrets.tf`,
`bootstrap/k3s-bootstrap/secrets.tf`) reads the real value at every
`plan`/`apply`, and Ansible's `pre_tasks`
(`infra/home-infra/ansible/playbooks/site.yml`) reads it at every
`site.yml` run -- moving any of those out to `pass`-only storage would
reintroduce exactly the manual-export step the SOPS-to-Secrets-Manager
cutover was built to remove. The real exceptions found so far:

- **`monitoring_grafana_admin_password`** -- **removed** 2026-09-10
  (Category I above). Not moved to local storage: deleted outright.
- **`shared_ingress_auth_password`/`_hash`** -- stays in Secrets
  Manager for now (Terraform actively keeps the Kubernetes Secret
  populated so the rollback stays a one-chain-edit swap, no rebuild),
  but it's a real trade-off worth revisiting if the underlying
  Authelia storage issue (`BACKLOG.md`) goes long enough without
  recurring that the rollback path stops earning its keep.

No other secret is a plausible candidate for local-only storage --
every one either feeds a live Terraform/Ansible read, or (Category
D's other entry, `authelia_admin_password_hash`) genuinely needs to be
wherever the service that checks it against Julian's real login can
reach at deploy time.
