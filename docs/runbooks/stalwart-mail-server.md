# Runbook: Stalwart mail server -- manual settings and known limits

Status: **Live, 2026-09-19/20.** Covers `info@jkandler.de`/
`julian@jkandler.de` (plus the `admin@` service account), built on
[`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' `modules/stalwart` and `modules/bulwark`. This
runbook exists because several real settings live only in Stalwart's
own runtime database, not in Terraform -- see "Why these aren't in
Terraform" below.

## Architecture, in brief

- **Stalwart** (`stalwart.jkandler.de`) is the actual mail server:
  SMTP inbound (port 25, relayed from the homeserver by
  `home-infra#51`'s `k3s_ingress_forward`), JMAP/DAV, and its own
  admin (`/admin`) and account (`/account`) web UIs.
- **Bulwark** (`mail.jkandler.de`) is a separate webmail client
  (`ghcr.io/bulwarkmail/webmail`), talking to Stalwart's JMAP endpoint
  directly from the browser.
- **Outbound mail** relays through Amazon SES ([`ses-relay`](https://github.com/KandlerLi/ses-relay)'s
  domain identity and IAM credential, shared with Alertmanager and
  Authelia), not direct-to-MX delivery from the k3s VM's own
  unproven IP.
- **Both `mail.jkandler.de` and `stalwart.jkandler.de`** sit behind
  Authelia forward-auth (`two_factor` policy) at the Traefik layer
  (`modules/ingress`) -- except Stalwart's own protocol paths
  (`/jmap`, `/dav`, `/.well-known`, `/auth`, autodiscover), which
  bypass it since JMAP/DAV/mail clients can't follow a login redirect.

## Why these aren't in Terraform

Stalwart's setup wizard was run once, choosing "internal directory"
and "manual DNS management" -- from that point on, Stalwart persists
its own runtime configuration (CORS, IP allow/block lists, MTA
outbound routes, the authentication directory) in its own database on
the PVC, not in a file this repo owns. There is no supported way to
manage these settings declaratively from outside Stalwart's own UI/API
without conflicting with what the UI itself writes (confirmed for MTA
routing specifically -- see Stalwart's own docs on mixing
`config.toml` and webadmin-managed settings). So these are tracked
here as manual settings, the same category as the FRITZ!Box's own
port-25 forward.

**Update 2026-09-25: most of these are now Terraform-managed.** The
community `tahacodes/stalwart` provider (pinned 0.2.4, in
[`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' `modules/stalwart_config`) talks to Stalwart's
management API, which is the same store the UI writes, so the conflict
described above doesn't apply. Managed there now: CORS and the allowed
Pod-network IP (`k3s-apps#81`), the ACME provider and the mail domain
with DKIM set to Manual (`#82`), and the SES relay route plus the
outbound routing strategy (`#90`, imported from the live settings with
no value changes). **Change these in Terraform, not in the UI**: a UI
edit shows up as drift in the next plan and gets reverted on apply.
Still manual on purpose: the Authentication Directory (the Authelia
OIDC switch below), because a mistake there locks out everything but
the admin API key. The sections below still describe what each
setting is and why; for the Terraform-managed ones, the UI steps are
only a fallback if the provider is ever unusable.

**Confirmed live pattern: Stalwart only reads several of these
settings at process start.** Changing CORS, the IP allow/block lists,
or the authentication directory in the UI does **not** take effect
until the Pod is restarted:

```bash
export KUBECONFIG=~/.kube/k3s-node-1.yaml
kubectl -n apps delete pod -l app=stalwart
```

This is a plain restart, not a config change -- mail data and
settings are on the PVC and survive it.

## Manual settings, and how to redo each

### CORS (`Settings -> ... -> Permissive CORS policy`)
*Terraform-managed (`modules/stalwart_config`) -- see the update above.*

Enabled, so Bulwark's browser-side JMAP calls to `stalwart.jkandler.de`
from `mail.jkandler.de` aren't blocked. `allow-origin: *`, not scoped
to Bulwark's own origin specifically -- Stalwart's UI only offers an
on/off toggle here, not a per-origin allowlist. Real exposure is
limited: every JMAP call still needs valid Stalwart credentials.

### Allowed IPs (`Security -> Allowed IPs`)
*Terraform-managed (`modules/stalwart_config`) -- see the update above.*

`10.42.0.0/16` (the k3s Pod network) is on the allow-list, so
Stalwart's own automatic IP-banning never blocks the Traefik Pod
itself (confirmed live: it did, once, and took down every real visitor
at the same time -- Traefik is the only IP Stalwart's HTTP listener
ever actually sees).

### Outbound SMTP relay (`MTA -> Outbound -> Routes` / `Strategy` / `TLS Strategies`)
*Terraform-managed (`modules/stalwart_config`) -- see the update above.*

- **Routes**: a `ses-relay` route, type Relay Host, to
  `email-smtp.eu-central-1.amazonaws.com:587`, SMTP, STARTTLS (not
  implicit TLS). Username: the SES SMTP username (`terraform output
  -raw smtp_username` in [`ses-relay`](https://github.com/KandlerLi/ses-relay)). Password: "Secret read from
  environment variable", `SES_SMTP_PASSWORD` -- populated by
  `modules/stalwart`'s own `SES_SMTP_USERNAME`/`SES_SMTP_PASSWORD` env
  vars (sourced from Terraform/Secrets Manager, not typed into the UI
  -- see that module's own comment for why).
- **Strategy -> Routing**: `ELSE 'ses-relay'` (was `'mx'` by default).
  `IF is_local_domain(rcpt_domain) THEN 'local'` unchanged.
- **TLS Strategies**: unchanged, `default` (DANE/MTA-STS/STARTTLS all
  `Optional`) works fine for a relay -- no new strategy needed
  (confirmed live).
- **DKIM Signatures** (`Email -> DKIM` or similar): **deleted**, both
  the wizard-generated RSA and Ed25519 keys. They were never published
  in DNS (the wizard's "Generate Email Signing Keys" toggle doesn't
  publish anything under "manual DNS management"), and signing twice
  -- once by Stalwart, once by SES's own Easy DKIM -- made SES reject
  the message outright (`554 ... Duplicate header 'DKIM-Signature'`).
  SES's own Easy DKIM ([`ses-relay`](https://github.com/KandlerLi/ses-relay)'s three DKIM CNAME records)
  covers `jkandler.de` DKIM signing; don't re-create a Stalwart-side
  DKIM signature unless SES relaying is removed entirely.

### Authentication Directory (`Authentication -> General`) -- the SSO/admin trade-off
An OIDC directory ("Authelia SSO": issuer `https://auth.jkandler.de`,
username claim `preferred_username`, **no** username domain --
confirmed live both `julian`/`admin` Stalwart accounts are named
plainly, and the account's own Email Address field
(`username@domain`, e.g. `julian@jkandler.de`) is what actually
matches the OIDC-resolved identity, not the bare Username field --
setting a username domain here would be redundant, not required)
lets Bulwark's own "Sign in with Authelia" button log straight into
the real mailbox via an OAuth bearer token, with no separate Stalwart
password.

**This field is a single, global switch, not additive.** Confirmed
live: while it points at the OIDC directory, Stalwart's own `/admin`
login screen is still password-only (no SSO button there) and the
internal password check is disabled entirely -- `admin@` cannot log
into the visual admin console at all while this is set.

**Left set to "Authelia SSO" permanently** (mailbox convenience over
console convenience) -- **but this doesn't mean losing admin access.**
Confirmed live 2026-09-20: an **API key** credential (`admin`'s own
`/account` -> Security -> API Keys, "for programmatic access") works
as a Bearer token against Stalwart's management API
(`/api/...`) **regardless of the Authentication Directory setting** --
it's validated as a Stalwart-internal credential, not routed through
the configured directory at all. This is the real, ongoing way to
administer Stalwart while SSO stays on:

```bash
export KUBECONFIG=~/.kube/k3s-node-1.yaml
# open the SSH tunnel to the k3s apiserver if it isn't already
ssh -f -N -L 6443:192.168.101.10:6443 julian@192.168.178.100
kubectl -n apps port-forward svc/stalwart-internal 18083:8080 &

apikey=$(pass show mail/admin/api)   # or wherever it's kept
curl -s -X POST http://127.0.0.1:18083/jmap/ \
  -H "Authorization: Bearer $apikey" -H 'Content-Type: application/json' \
  -d '{"using":["urn:ietf:params:jmap:core","urn:stalwart:jmap"],
       "methodCalls":[["x:Authentication/get",{"ids":["singleton"]},"c1"]]}'
unset apikey
```

The management API is JMAP, `POST /jmap/` (the `apiUrl` from
`/jmap/session`) with the `urn:stalwart:jmap` capability and `x:`
object types (`x:Authentication`, singleton id `singleton`). `/api`
is *not* the endpoint for these calls (POST `/api` returned 404,
confirmed live 2026-09-21), although `GET /api/account` does exist.
The session document doesn't list `urn:stalwart:jmap`, but the calls
work anyway as long as the key's account holds the `sys*`
permissions (`sysAuthenticationGet`/`sysAuthenticationUpdate` etc.).

The key was generated once (2026-09-20) and is kept in `pass` as
`mail/admin/api` -- outside Terraform/Secrets Manager entirely, same
as every other Stalwart account credential (nothing here reads it
back programmatically). Per Stalwart's own docs, this key **only**
works against `/api/...` (provisioning/config); it does not grant
mail-protocol login (IMAP/JMAP mail/SMTP) or the visual `/admin` UI
itself.

**Getting the visual UI back while SSO is on** (only for settings
that genuinely need it -- the toggle lives *inside* the UI, so with
SSO active you have no session to flip it from; use the API key):

1. Note the SSO directory's id: `x:Authentication/get` above returns
   `directoryId` (it was `jftufl1kaaqc` on 2026-09-21).
2. Switch to internal: `x:Authentication/set` with
   `{"update":{"singleton":{"directoryId":null}}}`, then
   `kubectl -n apps delete pod -l app=stalwart` (settings only load at start).
3. Log into `/admin` with `admin@`'s own Stalwart password, do the
   change.
4. Switch back: the same `set` with
   `{"directoryId":"<the id from step 1>"}`, restart the Pod again.
   Bulwark's SSO is down between steps 2 and 4.

Confirmed live 2026-09-21 that the API key carries
`sysAuthenticationUpdate` and the `get` works; the `set` call is what
the Stalwart docs specify for this object.

**If ever truly locked out of both** (API key revoked/lost *and* the
directory field stuck on OIDC): there is no known UI/API recovery path
from outside the server itself. Recovery would mean editing Stalwart's
own database file directly on the PVC (`kubectl exec` into the Pod, or
a debug Pod mounting the same PVC) -- not yet exercised or documented
in detail; treat as an unverified last resort. Keeping at least one
valid, saved API key at all times avoids ever reaching this.

### Authelia's `admin` identity
A second, separate Authelia user (`modules/authelia`'s
`authelia_stalwart_admin_password_hash`), purely so Stalwart's own
`admin` account has *an* Authelia identity to match against once SSO
is active -- it doesn't currently grant it into `/admin` (see above),
only into anything else gated the same way `julian`'s identity is.
Its own password lives only in Julian's password manager, same as
`julian`'s.

## Known limits / not yet done

- **Mail clients use IMAP (993) and submission (465)** with an app
  password, public since `home-infra#53`/`k3s-apps#79`. Stalwart serves
  its own Let's Encrypt certificate for `stalwart.jkandler.de` on both
  (`k3s-apps#82`). Receiving and sending from a client confirmed
  2026-10-11. Under SSO nobody can log into `/account`, so creating an
  app password means switching the directory to internal briefly.
- **No backup MX.** A homeserver outage delays inbound mail (senders
  retry for days) but doesn't lose it; short-retry automated senders
  could still be missed. Deliberately not built yet -- see chat/decision
  log around 2026-09-19/20 for the SES-inbound-backup design if this
  is revisited.
- **SPF hard-fails (`-all`)** since 2026-10-11 ([`ses-relay`](https://github.com/KandlerLi/ses-relay)): only SES
  and the homeserver may send as `jkandler.de`. A new sending path (another
  relay, a newsletter service) has to be added to the record first, or its
  mail fails SPF.
- **SES production access was granted 2026-09-21** (case
  178988954600742: out of the sandbox in eu-central-1, 50,000
  messages/day, 14/s). Confirmed 2026-10-11: a mail from a mail client
  through Stalwart reached an external recipient that is not a verified
  SES identity. Watch SES bounce/complaint rates -- AWS expects a process
  for them.
- **No off-server backup of mail data.** It's on the k3s VM's
  `local-path` PVC only.
