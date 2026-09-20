# Runbook: Stalwart mail server -- manual settings and known limits

Status: **Live, 2026-09-19/20.** Covers `info@jkandler.de`/
`julian@jkandler.de` (plus the `admin@` service account), built on
`infra/k3s-apps`' `modules/stalwart` and `modules/bulwark`. This
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
- **Outbound mail** relays through Amazon SES (`aws/ses-relay`'s
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

**Confirmed live pattern: Stalwart only reads several of these
settings at process start.** Changing CORS, the IP allow/block lists,
or the authentication directory in the UI does **not** take effect
until the Pod is restarted:

```bash
export KUBECONFIG=~/.kube/k3s-node-1.yaml
kubectl delete pod -l app=stalwart
```

This is a plain restart, not a config change -- mail data and
settings are on the PVC and survive it.

## Manual settings, and how to redo each

### CORS (`Settings -> ... -> Permissive CORS policy`)
Enabled, so Bulwark's browser-side JMAP calls to `stalwart.jkandler.de`
from `mail.jkandler.de` aren't blocked. `allow-origin: *`, not scoped
to Bulwark's own origin specifically -- Stalwart's UI only offers an
on/off toggle here, not a per-origin allowlist. Real exposure is
limited: every JMAP call still needs valid Stalwart credentials.

### Allowed IPs (`Security -> Allowed IPs`)
`10.42.0.0/16` (the k3s Pod network) is on the allow-list, so
Stalwart's own automatic IP-banning never blocks the Traefik Pod
itself (confirmed live: it did, once, and took down every real visitor
at the same time -- Traefik is the only IP Stalwart's HTTP listener
ever actually sees).

### Outbound SMTP relay (`MTA -> Outbound -> Routes` / `Strategy` / `TLS Strategies`)
- **Routes**: a `ses-relay` route, type Relay Host, to
  `email-smtp.eu-central-1.amazonaws.com:587`, SMTP, STARTTLS (not
  implicit TLS). Username: the SES SMTP username (`terraform output
  -raw smtp_username` in `aws/ses-relay`). Password: "Secret read from
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
  SES's own Easy DKIM (`aws/ses-relay`'s three DKIM CNAME records)
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
pgrep -f "ssh.*-L 6443:192.168.101.10:6443" \
  || ssh -f -N -L 6443:192.168.101.10:6443 julian@192.168.178.100
kubectl port-forward svc/stalwart-internal 18083:8080 &

apikey=$(pass show mail/admin/api)   # or wherever it's kept
curl -s -H "Authorization: Bearer $apikey" \
  http://127.0.0.1:18083/api/... 
```

The key was generated once (2026-09-20) and is kept in `pass` as
`mail/admin/api` -- outside Terraform/Secrets Manager entirely, same
as every other Stalwart account credential (nothing here reads it
back programmatically). Per Stalwart's own docs, this key **only**
works against `/api/...` (provisioning/config); it does not grant
mail-protocol login (IMAP/JMAP mail/SMTP) or the visual `/admin` UI
itself.

**Fallback, only for settings that genuinely need the visual UI**
(the API key doesn't help there): toggle **Authentication Directory**
back to empty, restart the Pod, log in with `admin@`'s own password,
do the change, then set it back to "Authelia SSO" and restart again.
Confirmed live this reverts and re-applies cleanly each time.

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

- **IMAP (993) and submission (465) aren't public.** No certificate
  strategy chosen for them yet -- needed before Thunderbird/phone
  clients can connect. Port 25 is the only public mail port today
  (`home-infra#51`, `k3s-apps#68`/`stalwart-smtp` Service).
- **No backup MX.** A homeserver outage delays inbound mail (senders
  retry for days) but doesn't lose it; short-retry automated senders
  could still be missed. Deliberately not built yet -- see chat/decision
  log around 2026-09-19/20 for the SES-inbound-backup design if this
  is revisited.
- **SPF is soft-fail (`~all`)**, deliberately, while outbound is new.
  Tighten to `-all` in `aws/ses-relay` once outbound has run cleanly
  for a week or two.
- **SES account is still in the sandbox** (production access
  requested 2026-09-19/20, case pending/approved -- check current
  status). Outbound to unverified recipients doesn't work until
  approved.
- **No off-server backup of mail data.** It's on the k3s VM's
  `local-path` PVC only.
