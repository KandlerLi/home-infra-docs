# ADR 0020: Network-wide DNS ad-blocking with Blocky

- Status: Accepted
- Date: 2026-08-28
- Deployment status: built, not yet applied -- `ansible/roles/blocky`
  written and unit-tested; `blocky_enabled` stays `false` (the role's own
  default) until the user validates it live and persists it in
  inventory, the same rollout discipline `deluge`/`monitoring` already
  followed

## Context

The user asked for network-wide DNS ad-blocking with a dashboard
comparable to Pi-hole's, specifically via Blocky rather than Pi-hole or
AdGuard Home.

## Decisions

**Blocky over Pi-hole/AdGuard Home**, per the user's own request -- not
re-litigated here. Worth recording since it drives the rest of this ADR:
Blocky ships **no web UI of its own**, unlike either alternative. It's a
single Go binary, YAML-configured, exposing Prometheus metrics and a
small REST API and nothing else visual.

**The dashboard is Grafana, fed by two sources, not one.** Asked
directly (dashboard depth vs. DNS rollout mechanism were the two
decisions that changed scope): the user chose the full Pi-hole-style
query log (top clients, top blocked domains, recent queries) over
aggregate-stats-only. That requires Blocky's native
`queryLog: type: postgresql` writer, so both are built:

- Blocky's own Prometheus metrics, scraped by the existing `monitoring`
  role (no new exporter needed -- unlike node_exporter/cAdvisor/
  blackbox_exporter, Blocky exposes real metrics natively) -- feeds the
  community "Blocky" Grafana dashboard (id 13768), which also gives a
  live blocking enable/disable toggle the query-log dashboard can't.
- A new PostgreSQL container, added purely to hold Blocky's query log --
  the first database this repo has ever had to run itself (Nextcloud AIO
  manages its own internally). Feeds the community "Blocky-Postgres"
  dashboard (id 17996 / `shizunge/blocky-postgresql-dashboard`), read
  through a new Grafana Postgres datasource, not through Prometheus.

Both dashboards were fetched from Grafana's own public dashboard API at
implementation time and adapted (datasource UID placeholders resolved to
this repo's real datasource uids; the `${VAR_BLOCKY_URL}` constant
resolved to `http://127.0.0.1:4000`), not written from scratch or from
memory.

**Two containers, one role** (`ansible/roles/blocky`) -- the same
"tightly coupled, one role" shape `nextcloud_aio` and `monitoring`
already use, since the Postgres instance exists only to serve Blocky's
own query log.

**`network_mode: host`, DNS bound to the host's real LAN IP** (auto-
discovered via `ansible_default_ipv4.address`, not hardcoded) -- the one
deliberate exception to this repo's usual "opt-in services stay
loopback-only until shared_ingress publishes them" pattern. DNS
inherently needs to be reachable by every device on the network from day
one; a loopback-published port can't provide that. Confirmed live before
committing to this: nothing else was already listening on port 53 on
that IP (only libvirt's own dnsmasq, bound to the isolated runner-VM
network's own address, a different IP entirely).

**`cap_drop: ALL` + `capabilities: [NET_BIND_SERVICE]` for Blocky
itself** -- the same shape `shared_ingress` (Traefik) already uses for
its own privileged ports 80/443, the minimal capability needed to bind
port 53 without a root uid.

**Postgres does *not* get the same capability treatment, and doesn't run
as a mapped host uid like most containers in this repo.** The official
postgres image's entrypoint needs to start as root to `gosu`/`chown`
into its own internal postgres user on first run; `cap_drop: ALL` or a
forced non-root uid would break that startup sequence. Its actual
security boundary is `listen_addresses=127.0.0.1` set explicitly via its
`command` -- without that, sharing Blocky's host network namespace would
otherwise expose it on every host interface, including the LAN-facing
one, by Postgres's own default.

**Static IP upstreams and bootstrap resolvers, never hostnames.**
Confirmed via Blocky's own current release notes that this is a real,
live concern (not hypothetical): a DNS server that will become the
network's *only* resolver can't depend on hostname resolution to reach
its own upstreams -- that's a chicken-and-egg problem worth designing
away entirely rather than trusting retry logic to paper over.

**Postgres data on `/mnt/black-hdd`, not the root disk**, matching
`monitoring`'s own reasoning for its Prometheus TSDB -- the root disk had
only ~19 GiB free at the time (tighter than when ADR 0019 was written).
The path is a single overridable variable (`blocky_postgres_data_dir`),
deliberately not hardcoded, so moving it to the root SSD later (traded
against the HDD's slower-but-endurance-free I/O vs. the SSD's faster
I/O on a genuinely small, continuously-written, retention-capped
dataset) is a one-line change, not a role rewrite.

## Consequences

- Enabling the role alone changes nothing for any device on the network
  -- they keep using whatever DNS they already have. Actually routing
  traffic through Blocky needs one manual step Ansible can't do:
  pointing the FRITZ!Box's own upstream DNS at the homeserver (see the
  top-level README) -- the standard mechanism keeps the FRITZ!Box itself
  as the DHCP-advertised DNS server for every device (so its own local-
  hostname resolution for other LAN devices keeps working), while the
  FRITZ!Box's own lookups route through Blocky.
- `dnsutils` (for `dig`, used only by the role's own live DNS-answering
  check) is installed scoped to this role, not added to `base` -- the
  same precedent `sankey_export`'s Playwright/Chromium dependency
  already set for a role-specific tool that doesn't belong in every
  host's baseline.
- The `monitoring` role's Grafana datasource provisioning file was
  renamed from `prometheus.yml` to `datasources.yml` now that it can
  carry more than one datasource -- with an explicit cleanup task
  removing the old file, not just a silent stop-writing-it, so an
  already-deployed homeserver doesn't keep a stale file around forever.
