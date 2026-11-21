# ADR 0034: Euro-Office on k3s, rootless, behind forward-auth

- Status: Accepted (2026-11-21)
- Date: 2026-11-21
- Deployment status: live since 2026-11-21; opening, editing and saving a document confirmed by Julian
- Follows: [ADR 0033](0033-nextcloud-on-k3s.md), which deferred the document editor

## Context

Nextcloud moved to k3s on 2026-11-21 (ADR 0033) without a document editor.
AIO's Euro-Office image runs `supervisord` and `nginx` as root with its own
PostgreSQL, RabbitMQ and Redis; `apps` enforces PodSecurity `restricted`.

Options considered:

- **Euro-Office's Kubernetes build** (`ghcr.io/euro-office/cluster-docs`,
  v9.3.8, from `Euro-Office/DocumentServer`'s `orchestrated` Dockerfile and
  the `Euro-Office/Kubernetes-Docs` chart): one image, run as `ds`
  (uid 101) in the roles docservice, converter and proxy (nginx on 8888);
  PostgreSQL, Redis and RabbitMQ are external. Same editor as under AIO.
  Upstream still calls its Docker images testing/integration builds.
- **Collabora (CODE)**: one container, less memory, but a different
  (LibreOffice-based) editor, and running it without capabilities relies on
  the Landlock sandbox that is new in 26.04.3.
- **No editor**: nothing to run; no in-browser editing.

## Decision

Run Euro-Office's Kubernetes build in `apps` as plain manifests under
`k3s-apps/argocd/eurooffice`, digest-pinned, without Helm: the chart's
core only.

- **One Pod**: proxy, docservice and converter from `cluster-docs`, all
  rootless as 101; PostgreSQL, Redis and RabbitMQ as native sidecars on
  localhost. An init container copies the schema from `cluster-utils`, and
  PostgreSQL applies it on its first start.
- **No persistent storage.** Euro-Office only holds open editing sessions;
  documents are always saved back to Nextcloud. A restart ends open
  sessions at most (AIO's data volume held 20 bytes).
- **JWT** between Nextcloud and Euro-Office, the secret in Secrets Manager
  (`k3s-apps/eurooffice`), read by both through ESO; Nextcloud's start
  hook configures the `eurooffice` connector app.
- **Paths**: browsers load the editor from
  `https://nextcloud.jkandler.de/eurooffice/` (Traefik strips the prefix).
  Nextcloud calls Euro-Office in-cluster on `eurooffice-svc`. Euro-Office
  fetches documents and calls back through Nextcloud's public route,
  pinned in the Pod to `ingress-svc`'s ClusterIP with `hostAliases`, like
  Vaultwarden; its request filter allows exactly that address.
- **Forward-auth on the editor route** (Julian, 2026-11-21): only browsers
  with an Authelia session reach it. No new `forward_auth: false`
  exception.
- **NetworkPolicy**: in from Traefik and Nextcloud only, out to DNS and
  Traefik only.

## Consequences

- In-browser editing of docx/xlsx/pptx/odf works again, with the same
  editor as under AIO.
- Editing from the phone app and through public share links doesn't work:
  those have no Authelia session. Changing that later means a reviewed
  `forward_auth: false` route.
- About 1.3–1.5 GiB more memory on `k3s-node-1`.
- Upstream calls the images testing builds; Renovate bumps the digest, and
  a broken release only takes editing down, not Nextcloud.
- No backup needed: Euro-Office holds no state worth keeping.

## Implementation notes

Found while rolling out (2026-11-21):

- **The image doesn't set `COMPANY_NAME`**, only `COMPANY_NAME_LOW`,
  while the entrypoints build every path from it; the Deployment sets it.
- **No body limit on the editor route.** `max_body_bytes` adds Traefik's
  buffering middleware, which breaks the editor's socket.io websocket.
  Euro-Office's own nginx caps bodies at 100m.
- **`X-Forwarded-Proto: https` is forced on the route.** Traefik sends
  `wss` on websocket upgrades; Euro-Office's nginx then falls back to
  `http`, and docservice builds the `Editor.bin` link from that socket's
  headers, so the browser blocked it as mixed content ("Download failed").
  AIO's old Nextcloud route forced the header for the same reason.
- **`SECURE_LINK_SECRET` is set**: its default is a fixed public string.
- The Pod uses about 550 MiB idle, less than estimated.
