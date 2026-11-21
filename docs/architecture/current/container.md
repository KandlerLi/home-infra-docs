# Home Infrastructure — Container Diagram

- **C4 type:** Container
- **Scope:** Home Infrastructure software system
- **Status:** Current as of 2026-11-21 (Nextcloud on k3s, ADR 0033)

Grouped by where each container runs. Details per hostname and auth:
[`current-state.md`](../../current-state.md).

```mermaid
flowchart TB
    julian["Julian / remote clients<br/>[Person]"]
    ses["AWS SES<br/>[External Software System]"]

    subgraph k3s["k3s cluster (k3s-node-1)"]
        traefik["Traefik<br/>[Container: Traefik v3, file provider]<br/>TLS, routing, rate limits, security headers"]
        authelia["Authelia<br/>[Container]<br/>Forward-auth, OIDC provider, TOTP/WebAuthn"]
        grafana["Grafana<br/>[Container]<br/>Dashboards"]
        alertmanager["Alertmanager<br/>[Container]<br/>Alert routing"]
        blocky["Blocky<br/>[Container]<br/>LAN DNS with ad blocking"]
        blockydb[("Blocky query log<br/>[Container: PostgreSQL]")]
        stalwart["Stalwart<br/>[Container]<br/>SMTP, IMAP, JMAP, CalDAV/CardDAV"]
        bulwark["Bulwark<br/>[Container]<br/>Webmail"]
        paperless["Paperless-ngx<br/>[Container]<br/>Document archive with OCR"]
        paperlessdb[("Paperless DB<br/>[Container: PostgreSQL + Redis]")]
        vaultwarden["Vaultwarden<br/>[Container]<br/>Password manager, SQLite"]
        alloyk["Alloy<br/>[Container]<br/>Pod logs via the API"]
        other["Landing page, Headlamp,<br/>Argo CD, sankey_export<br/>[Containers]"]
        aio["Nextcloud<br/>[Container: k3s Pod]<br/>Files, calendar, contacts; PostgreSQL, Redis, notify_push"]
    end

    subgraph host["Homeserver (Debian)"]
        prometheus["Prometheus + exporters<br/>[Container: Docker]<br/>Metrics and probes"]
        loki[("Loki + Alloy<br/>[Container: Docker]<br/>Host journal and Pod logs")]
        ntfy["ntfy-relay<br/>[Container: systemd]<br/>Alertmanager to ntfy"]
        black[("black-hdd<br/>[Data store: ext4 + NFS exports]<br/>Live data")]
        red[("red-hdd<br/>[Data store]<br/>Backups")]
    end

    julian -->|"HTTPS"| traefik
    julian -->|"SMTP/IMAP"| stalwart
    traefik -->|"Forward-auth check"| authelia
    traefik --> grafana & bulwark & paperless & vaultwarden & other & aio & stalwart
    bulwark -->|"JMAP"| stalwart
    stalwart -->|"Outbound relay"| ses
    grafana & paperless & bulwark & aio -->|"OIDC login"| authelia
    other -->|"WebDAV (sankey_export)"| aio
    vaultwarden -->|"OIDC via Traefik ClusterIP"| traefik
    blocky --> blockydb
    paperless --> paperlessdb
    prometheus -->|"Alerts"| alertmanager
    alertmanager -->|"Webhook"| ntfy
    grafana -->|"Queries"| prometheus
    grafana -->|"Log queries"| loki
    alloyk -->|"Push"| loki
    aio -->|"NFS data directory"| black
    paperless -->|"NFS media"| black
    aio -->|"pg_dump CronJob; restic on the host"| red

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef container fill:#438dd5,color:#fff,stroke:#0b4884
    classDef datastore fill:#2e8b57,color:#fff,stroke:#1d5c3a
    class julian person
    class ses external
    class traefik,authelia,grafana,alertmanager,alloyk,blocky,stalwart,bulwark,paperless,vaultwarden,other,aio,prometheus,ntfy container
    class blockydb,paperlessdb,loki,black,red datastore
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
