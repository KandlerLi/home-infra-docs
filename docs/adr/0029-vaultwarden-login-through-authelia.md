# ADR 0029: Vaultwarden logs in through Authelia only (SSO)

- Status: Accepted
- Date: 2026-11-10
- Amends: [ADR 0028](0028-vaultwarden-public-without-forward-auth.md)
- Deployment status: live since 2026-11-10 (`k3s-apps#161`, secret container `secrets-manager#47`, CI and operator read access `repo-infra#66`, `terraform-state#31`); verified from outside: a password grant is rejected with "SSO sign-in is required"

## Context

ADR 0028 kept Vaultwarden's own login: email and master password at
`/identity`, with Vaultwarden's own TOTP as the second factor. Once the
client API is public, that login can be tried from the internet, and
2FA lives in a second place next to Authelia.

Vaultwarden 1.37 supports OpenID Connect login, and Authelia is already
the OIDC provider for Grafana, Open WebUI, Nextcloud, Bulwark and
Paperless.

## Decision

Vaultwarden logs in **only through Authelia** (`SSO_ENABLED`,
`SSO_ONLY`):

- Authelia client `vaultwarden` with the `julian_only` policy
  (two-factor), PKCE, and refresh tokens so the apps keep their session.
  `julian_only` matters here: the Stalwart `admin` identity shares
  Julian's email address and must not be able to sign in or sign up.
- The first SSO login creates the account. Password signups are off for
  good, and SSO signups close when `initial_signup` is flipped, together
  with opening the client API as ADR 0028 describes.
- **The master password stays.** SSO replaces the login, not the
  encryption: the vault is still decrypted client-side with the master
  password, which neither Authelia nor the server ever sees.
- Vaultwarden reaches Authelia in-cluster: `auth.jkandler.de` is pinned
  to Traefik's ClusterIP, so ADR 0028's "no internet egress" still
  holds. Egress is cluster DNS and Traefik on 443.

## Consequences

- The public API no longer accepts a password login, so the master
  password cannot be guessed online. Online guessing now targets
  Authelia, which has its own regulation and two factors.
- 2FA is managed in one place, Authelia. Vaultwarden's own 2FA is not
  needed.
- Logging in on a new device needs Authelia. Devices already logged in
  keep unlocking locally with the master password while Authelia is
  down.
- Losing Authelia's second factor locks Vaultwarden logins too. The
  recovery path is Authelia's: an admin resets the factor in Authelia's
  storage. Turning `SSO_ONLY` off is the fallback when Authelia itself
  cannot be repaired.
- SSO is newer in Vaultwarden than its password login. The test
  accounts check that every client (web, browser extension, desktop,
  iOS) handles it before real accounts move in.
