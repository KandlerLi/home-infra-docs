# Home Infrastructure — Container Diagram

- **C4 type:** Container
- **Scope:** Home Infrastructure software system
- **Status:** Current as of 2026-10-11

Grouped by where each container runs. Details per hostname and auth:
[`current-state.md`](../../current-state.md).

```mermaid
flowchart TB
    julian["Julian / remote clients<br/>[Person]"]
    anthropic["Anthropic API<br/>[External Software System]"]
    ses["AWS SES<br/>[External Software System]"]

    subgraph k3s["k3s cluster (k3s-node-1)"]
        traefik["Traefik<br/>[Container: Traefik v3, file provider]<br/>TLS, routing, rate limits, security headers"]
        authelia["Authelia<br/>[Container]<br/>Forward-auth, OIDC provider, TOTP/WebAuthn"]
        owui["Open WebUI<br/>[Container]<br/>Chat UI"]
        agent["home_agent<br/>[Container: Python]<br/>Bounded API, model and tool loop"]
        nctools["nextcloud_tools<br/>[Container: Python sidecar]<br/>WebDAV/OCS tools for one Nextcloud account"]
        grafana["Grafana<br/>[Container]<br/>Dashboards"]
        alertmanager["Alertmanager<br/>[Container]<br/>Alert routing"]
        blocky["Blocky<br/>[Container]<br/>LAN DNS with ad blocking"]
        blockydb[("Blocky query log<br/>[Container: PostgreSQL]")]
        stalwart["Stalwart<br/>[Container]<br/>SMTP, IMAP, JMAP, CalDAV/CardDAV"]
        bulwark["Bulwark<br/>[Container]<br/>Webmail"]
        paperless["Paperless-ngx<br/>[Container]<br/>Document archive with OCR"]
        paperlessdb[("Paperless DB<br/>[Container: PostgreSQL + Redis]")]
        other["qBittorrent, landing page,<br/>Headlamp, sankey_export<br/>[Containers]"]
    end

    subgraph host["Homeserver (Debian)"]
        aio["Nextcloud AIO<br/>[Container: Docker]<br/>Files, calendar, Talk, Collabora"]
        hometools["home-tools<br/>[Container: systemd, Python]<br/>Read-only host facts"]
        prometheus["Prometheus + exporters<br/>[Container: Docker]<br/>Metrics and probes"]
        ntfy["ntfy-relay<br/>[Container: systemd]<br/>Alertmanager to ntfy"]
        black[("black-hdd<br/>[Data store: ext4 + NFS exports]<br/>Live data")]
        red[("red-hdd<br/>[Data store]<br/>Backups")]
    end

    julian -->|"HTTPS"| traefik
    julian -->|"SMTP/IMAP"| stalwart
    traefik -->|"Forward-auth check"| authelia
    traefik --> owui & agent & grafana & bulwark & paperless & other & aio & stalwart
    owui -->|"OpenAI-compatible chat"| agent
    agent -->|"Unix socket"| nctools
    agent -->|"HTTP 192.168.101.1:8095"| hometools
    agent -->|"Messages API"| anthropic
    nctools -->|"WebDAV/OCS on 192.168.101.1:11000"| aio
    bulwark -->|"JMAP"| stalwart
    stalwart -->|"Outbound relay"| ses
    owui & grafana & paperless & bulwark -->|"OIDC login"| authelia
    blocky --> blockydb
    paperless --> paperlessdb
    prometheus -->|"Alerts"| alertmanager
    alertmanager -->|"Webhook"| ntfy
    grafana -->|"Queries"| prometheus
    aio --> black
    paperless -->|"NFS media"| black
    aio -->|"Borg"| red

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef container fill:#438dd5,color:#fff,stroke:#0b4884
    classDef datastore fill:#2e8b57,color:#fff,stroke:#1d5c3a
    class julian person
    class anthropic,ses external
    class traefik,authelia,owui,agent,nctools,grafana,alertmanager,blocky,stalwart,bulwark,paperless,other,aio,hometools,prometheus,ntfy container
    class blockydb,paperlessdb,black,red datastore
```

Not drawn, to keep the view readable: the CI runners (`k3s-node-2`, see
[deployment](deployment.md)), Authelia's own storage (SQLite on a PVC,
sessions in Redis), Stalwart's RocksDB PVC, and the backup jobs (listed in
`current-state.md`).

## Legend

- Dark blue: person
- Grey: external system
- Medium blue: C4 container (application)
- Green: C4 container (data store)
- Unlabelled arrows from Traefik: HTTP routing to that service
