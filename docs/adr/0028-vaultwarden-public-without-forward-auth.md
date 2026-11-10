# ADR 0028: Vaultwarden on k3s, its client API public without Authelia forward-auth

- Status: Proposed
- Date: 2026-11-10
- Deployment status: not deployed. Prerequisite live: the ingress keeps the client IP (`k3s-apps#157`, confirmed 2026-11-10 with real public client addresses in Traefik's access log)

## Context

Website logins should move out of the browser into a password manager
that also works on the iPhone. Julian chose to self-host a
Bitwarden-compatible server, starting with a few test accounts.

The Bitwarden apps (desktop, browser extension, iOS) talk to the
server's API directly. They cannot complete an Authelia forward-auth
login in front of it. Every public route here sets
`forward_auth = true`, apart from apps that authenticate themselves.
This one needs to be such an exception.

Until `k3s-apps#157`, Traefik saw a pod IP for every request, so any
per-client limit was one bucket for the whole internet. That is fixed
now, and per-IP throttling is part of this design.

## Decision

Run **Vaultwarden** (pinned by digest) in `k3s-apps` at
`vault.jkandler.de`, and split its routes:

| Path | Route |
| --- | --- |
| `/api`, `/identity`, `/notifications/hub` | Public, `forward_auth = false`: what the apps need |
| `/admin` | Blocked at Traefik; the admin panel is also disabled (no `ADMIN_TOKEN`) |
| Everything else (web vault) | Behind Authelia forward-auth |

Hardening around the public part:

- **Login throttling per client IP** at Traefik (`rate_limit`) and in
  Vaultwarden's own login limiter, which reads Traefik's `X-Real-Ip`.
- **No self-service accounts.** Signups are open only for the first
  deploy, then switched off by a follow-up change. Invitations,
  organization creation, Sends, emergency access and password hints
  stay off.
- **Small feature surface.** Icon downloads are off, so the server makes
  no outbound requests for arbitrary domains. Attachments are off,
  because their download path is not on the public list. Push relay is
  off: apps sync on open and pull-to-refresh instead.
- **NetworkPolicy:** only Traefik reaches the Pod; no egress beyond
  cluster DNS.
- **Account-side:** TOTP 2FA and Argon2id KDF on the account, set by
  hand at first login. Vaultwarden cannot enforce them for a single user.

Storage is SQLite on a `local-path` volume, not NFS, which pins the Pod
to one node. A nightly CronJob writes an online SQLite backup, together
with the server's key files, to the homeserver. A password-protected
vault export is kept offline, off the homeserver.

Also considered:

- **Bitwarden Cloud (EU region):** no operations and stays available
  when the homeserver is down. Julian prefers self-hosting.
- **LAN/VPN only** via the FRITZ!Box WireGuard (ADR 0025): no public
  pre-auth surface. Rejected for day-to-day convenience. It stays the
  fallback if the public route turns out to be a problem, and needs only
  a route change.
- **Client certificates (mTLS):** strongest gate, but support across the
  Bitwarden apps, iOS in particular, is not reliable enough to depend on.
- **KeePassXC with Nextcloud sync:** no server, but no first-party iOS
  app and sync conflicts on a shared file.

## Consequences

- Vaultwarden's API is reachable from the internet before any login. A
  pre-auth vulnerability there would be exposed directly. Updates come
  through Renovate's digest PRs, and the feature surface above is kept
  small to limit what can be hit.
- The vault is end-to-end encrypted with the master password. A server
  compromise gives an attacker encrypted data to guess offline, so the
  master password's strength is the last line.
- Clients keep an encrypted copy and can read and autofill while the
  server is down, but cannot change entries or set up new devices.
- Infrastructure secrets (AWS keys, the Borg passphrase) stay in `pass`.
  Keeping the Borg passphrase out of a vault that Borg itself backs up
  avoids a restore that needs what it is restoring.
- TOTP seeds stay in a separate authenticator app, not in this vault,
  so one compromise doesn't give both factors.
- Rollback is removing the route, or limiting it to LAN/VPN. No other
  service depends on Vaultwarden.
