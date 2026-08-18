# Private Homeserver Agent — Proposed Deployment Diagram

- **C4 type:** Deployment
- **Scope:** Proposed production deployment, including authenticated ingress
- **Status:** Proposed; dashed elements and relationships are not deployed

```mermaid
flowchart TB
    user["Julian's Browser or API Client<br/>[Client Device]"]
    route53["AWS Route 53<br/>[Infrastructure Node]<br/>Future ai.jkandler.de record"]
    cloud["OpenAI API<br/>[External Software System]"]

    subgraph edge["Home Network Edge [Infrastructure Node]"]
        fritzbox["Fritzbox<br/>[Router/NAT/Firewall]"]
    end

    subgraph host["homeserver [Debian Deployment Node]"]
        proxy["Shared Reverse Proxy<br/>[Proposed Container Instance]<br/>HTTPS, authentication, routing, limits"]
        tools["home-tools.service<br/>[Proposed Container Instance]<br/>Unprivileged systemd service with fixed collectors"]
        socket["/run/home-tools<br/>[Infrastructure Node]<br/>Group-restricted Unix socket directory"]

        subgraph docker["Docker Engine [Execution Environment]"]
            agent["home-agent<br/>[Proposed Container Instance]<br/>Read-only filesystem, no capabilities, local API"]
            nextcloud["Nextcloud AIO Apache<br/>[Current Container Instance]<br/>Would move behind shared proxy"]
        end
    end

    user -.->|"Would resolve ai.jkandler.de"| route53
    user -.->|"Would connect to the resolved public IPv4 endpoint"| fritzbox
    fritzbox -.->|"Would forward TCP 443"| proxy
    proxy -.->|"Would authenticate and proxy ai.jkandler.de"| agent
    proxy -.->|"Would route the existing Nextcloud hostname"| nextcloud
    agent -.->|"Would call OpenAI Responses API over HTTPS"| cloud
    agent -.->|"Would connect to mounted socket directory"| socket
    socket -.->|"Would deliver fixed HTTP GET requests"| tools

    classDef client fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef node fill:#f5e6c8,color:#111,stroke:#9b6b21
    classDef proposedNode fill:#f5e6c8,color:#111,stroke:#9b6b21,stroke-dasharray:5 5
    classDef proposed fill:#85bbf0,color:#111,stroke:#0b4884,stroke-dasharray:5 5
    classDef current fill:#438dd5,color:#fff,stroke:#0b4884
    class user client
    class cloud external
    class route53,fritzbox node
    class socket proposedNode
    class proxy,tools,agent proposed
    class nextcloud current
```

## Important sequencing

The local agent milestone does not require the shared reverse proxy. Its API
would initially bind only to `127.0.0.1:8090`.

Publishing `ai.jkandler.de` is a later change because Nextcloud AIO currently
owns port 443. A safe public deployment requires:

1. configure Nextcloud AIO for supported reverse-proxy mode
2. place a shared reverse proxy on ports 80/443
3. preserve and validate the existing Nextcloud hostname first
4. add independent agent authentication and rate limits
5. create `ai.jkandler.de` DNS without adding an unverified AAAA record
6. expose the agent only after local and external security tests pass

## Legend

- Blue: proposed client
- Grey: external software system
- Beige: infrastructure/deployment node
- Light blue with dashed border: proposed container instance
- Solid blue: existing Nextcloud instance that would be migrated
- Dashed relationship: proposed/not deployed
