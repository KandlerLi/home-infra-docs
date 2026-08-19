# Public Home-Agent Routing — Deployment Diagram

- **C4 type:** Deployment
- **Scope:** Authenticated agent publication through shared ingress
- **Status:** Current and verified as of 2026-08-19

```mermaid
flowchart TB
    user["Julian's Browser or API Client<br/>[Client Device]"]
    route53["AWS Route 53<br/>[Infrastructure Node]<br/>ai.jkandler.de CNAME to jkandler.de"]
    cloud["OpenAI API<br/>[External Software System]"]

    subgraph edge["Home Network Edge [Infrastructure Node]"]
        fritzbox["Fritzbox<br/>[Router/NAT/Firewall]"]
    end

    subgraph host["homeserver [Debian Deployment Node]"]
        proxy["Traefik v3.7.1<br/>[Current Container Instance]<br/>ACME TLS and file routing; no Docker socket"]
        tools["home-tools.service<br/>[Current Container Instance]<br/>Unprivileged systemd service with fixed collectors"]
        socket["/run/home-tools<br/>[Current Infrastructure Node]<br/>Group-restricted Unix socket directory"]

        subgraph docker["Docker Engine [Execution Environment]"]
            agent["home-agent<br/>[Current Container Instance]<br/>Read-only filesystem, no capabilities, 127.0.0.1:8090"]
            nextcloud["Nextcloud AIO Apache<br/>[Current Container Instance]<br/>127.0.0.1:11000"]
        end
    end

    user -->|"Resolves ai.jkandler.de"| route53
    user -->|"Connects to the resolved public IPv4 endpoint"| fritzbox
    fritzbox -->|"Forwards TCP 443"| proxy
    proxy -->|"Applies Basic Auth and limits for ai.jkandler.de"| agent
    proxy -->|"Routes nextcloud.jkandler.de to 127.0.0.1:11000"| nextcloud
    agent -->|"Calls OpenAI Responses API over HTTPS"| cloud
    agent -->|"Connects to read-only mounted socket directory"| socket
    socket -->|"Delivers fixed HTTP GET requests"| tools

    classDef client fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef node fill:#f5e6c8,color:#111,stroke:#9b6b21
    classDef proposed fill:#85bbf0,color:#111,stroke:#0b4884,stroke-dasharray:5 5
    classDef current fill:#438dd5,color:#fff,stroke:#0b4884
    class user client
    class cloud external
    class route53,fritzbox node
    class socket node
    class proxy,tools,agent,nextcloud current
```

## Important sequencing

The local agent milestone is complete. Its API binds only to
`127.0.0.1:8090`.

Traefik owns ports 80/443, the Nextcloud migration is complete, and the agent
route is deployed with:

1. a preserved and verified Nextcloud hostname
2. independent agent authentication, rate limits, and request-size limits
3. `ai.jkandler.de` DNS without an AAAA record
4. successful local and external security tests

## Legend

- Blue: client
- Grey: external software system
- Beige: infrastructure/deployment node
- Solid blue: deployed proxy, agent, collector, and Nextcloud instances
- Solid relationship: deployed traffic flow
