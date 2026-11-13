# ADR 0031: Argo CD's UI public on argo.jkandler.de, behind Authelia twice

- Status: Accepted
- Date: 2026-11-13
- Deployment status: not deployed yet; supersedes the "core mode" part of
  [decisions](../decisions.md#argo-cd-trial-core-mode-one-app)

## Context

Argo CD deploys the landing page since the trial in `k3s-bootstrap#54`,
in core mode: no API server, no login, the UI only on the laptop through
the kubeconfig. Julian wants to run it properly, with its UI at
`argo.jkandler.de` and the Authelia login, still for the landing page
only.

Argo CD's UI can sync, roll back and delete what Argo CD manages, so it
is a deployment tool on a public hostname. Every public route here keeps
Authelia forward-auth unless the app authenticates itself (see the
public boundary in `current-state.md`).

## Decision

- **Forward-auth stays on.** The `argo` route in `k3s-apps` sets
  `forward_auth = true`, and an Authelia access rule admits only
  `user:julian` (two-factor) to that hostname.
- **Argo CD logs in through Authelia's OIDC as well**: client `argocd`,
  confidential with PKCE S256 (Argo CD v3.5's server-side flow sends it
  with `enablePKCEAuthentication`), policy `julian_only`. Both share the
  Authelia session, so this is one login in practice.
- **No other way in:** the local admin, Dex, anonymous access, exec and
  API keys stay off. Argo CD's RBAC gives `role:admin` to the `admins`
  group and nothing to anyone else.
- **Unchanged scope:** no ClusterRoles; Argo CD sees only `apps`, only
  ConfigMaps, Services and Deployments. The controller may change or
  delete only the landing page's three objects by name, and the server
  only reads them, so even a hijacked admin session reaches no further.
- **Secret:** the client secret's plaintext half lives in
  `home-infra/argocd` (read by `k3s-bootstrap-local`, granted in
  `terraform-state`), the hash in `home-infra/authelia`. It rotates with
  `k3s-apps`' `rotate-oidc-client-secret.sh argocd`.
- `auth.jkandler.de` is pinned to Traefik's ClusterIP in Argo CD's Pods,
  like Vaultwarden, so the OIDC discovery stays in-cluster.

## Alternatives

- **OIDC only, without forward-auth** (like Paperless): the CLI could log
  in over the route, but the login page and API would be public. Not
  worth it for a tool with deploy rights; the CLI uses the kubeconfig.
- **Stay in core mode:** no public surface at all, but the UI needs the
  CLI and an SSH tunnel every time.

## Consequences

- One more public hostname and blackbox probe (`http_2xx_or_401`).
- Anyone who gets both Julian's Authelia session and second factor can
  sync, roll back or delete the landing page; nothing else.
- Widening Argo CD to more apps is a separate decision: it widens the
  Role in `k3s-bootstrap`, and apps with secrets need their values in
  the cluster.
