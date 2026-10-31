# ADR 0026: qBittorrent replaces Deluge, so `apps` can enforce PodSecurity `restricted`

- Status: Accepted
- Date: 2026-10-31
- Deployment status: live since 2026-10-31 (`k3s-apps#142`); `apps` on `enforce: restricted` with `k3s-bootstrap#44`

## Context

`auth` and `monitoring` already enforced PodSecurity `restricted`;
`apps` stayed on `baseline`. The one blocker without a configuration
fix was Deluge: linuxserver's image starts as root (s6-overlay) and
only then drops to PUID/PGID, and no maintained Deluge image runs
non-root. Building our own image was the alternative.

Deluge also had no login fitting behind Authelia: Terraform computed
`sha1(salt)` to give it an empty password, so any Pod in the cluster
could use its Web UI.

## Decision

Replace it with **qBittorrent** (`ghcr.io/home-operations/qbittorrent`,
pinned by digest), which runs as any uid with a read-only root
filesystem. It runs as 993/986 on Deluge's NFS volumes, keeps
`torrent.jkandler.de` behind Authelia forward-auth and keeps the
peer port 6881.

Configuration is Terraform-authoritative: `qBittorrent.conf` comes from
a ConfigMap on every start. The image's own default whitelists every
private range and disables CSRF and Host-header checks; ours skips the
login only for the pod CIDR, keeps both checks and AutoRun off, and a
NetworkPolicy admits only Traefik to the Web UI port.

Also considered: Transmission (no maintained non-root image, sparse
Web UI) and Flood as a front end (an extra service for no gain here).
No torrent client supports OIDC itself, so forward-auth stays the
pattern.

## Consequences

- Settings changed in the Web UI are reset on the next restart;
  torrents themselves persist (SQLite resume data).
- qBittorrent logs a temporary admin password per session. It grants
  nothing: whitelisted callers skip the login, and nothing else
  reaches the port.
- The Web UI can start external programs (AutoRun, off by config). Its
  blast radius is the non-root, capability-free container.
- Storage keeps Deluge's names (`deluge-config`, `deluge-downloads`,
  host account `deluge`). Deluge's files stay next to qBittorrent's on
  `deluge-config` as a rollback path.
