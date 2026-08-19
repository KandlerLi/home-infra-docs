# Open WebUI Frontend — Deployment View

- **C4 type:** Deployment
- **Scope:** Private chat frontend and proposed publication at `ai.jkandler.de`
- **Status:** Private runtime deployed; public route not deployed as of 2026-08-19

```mermaid
flowchart LR
    person["Julian<br/>[Person]"]
    openai["OpenAI Responses API<br/>[External Software System]"]

    subgraph host["homeserver [Deployment Node: Debian]"]
        ingress["Shared ingress<br/>[Container: Traefik]<br/>TLS, route-specific auth and limits"]
        webui["Open WebUI<br/>[Container: v0.11.0-slim]<br/>Chat and account state"]
        agent["home-agent<br/>[Container: Python]<br/>Fixed prompt and tool policy"]
        tools["home-tools<br/>[systemd service]<br/>Fixed read-only collectors"]
        webui_data[("/var/lib/open-webui<br/>[Data store]")]
    end

    person -->|"HTTPS"| ingress
    ingress -.->|"Proposed catch-all route to 127.0.0.1:8091"| webui
    ingress -->|"Retained exact /healthz and /v1/chat routes to 127.0.0.1:8090"| agent
    webui -->|"OpenAI-compatible API over non-masqueraded Docker bridge"| agent
    webui -->|"Persists accounts and chats"| webui_data
    agent -->|"Responses API with server-held credential"| openai
    agent -->|"Fixed Unix-socket GET requests"| tools

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef current fill:#438dd5,color:#fff,stroke:#0b4884
    classDef proposed fill:#fff2cc,color:#111,stroke:#8a6d1d,stroke-dasharray: 5 5
    classDef datastore fill:#d5ead8,color:#111,stroke:#3b7044
    class person person
    class openai external
    class ingress,agent,webui,tools current
    class webui_data datastore
```

## Security boundaries

- Open WebUI receives only a non-secret placeholder key. The existing
  `home-agent` continues to hold the real provider credential and defines the
  server-side system prompt and allowed tools.
- Caller-provided system/developer messages are discarded by the compatibility
  API. Conversation length, message length, request bytes, tool calls, and
  model rounds remain bounded.
- Open WebUI has no Docker socket, host filesystem access, shell, code
  interpreter, plugin installation, file upload, web search, or workspace tool
  access. Its non-masqueraded Docker bridge reaches `home-agent`, permits the
  loopback publication, and blocks external egress.
- The public UI uses Open WebUI's own account login. HTTP Basic Auth cannot be
  stacked on that route because both mechanisms need the `Authorization`
  header. The retained direct API routes continue to use Traefik Basic Auth,
  and the first Open WebUI administrator is created before publication.
- Open WebUI account and chat data is new sensitive state. It is kept in
  `/var/lib/open-webui` and must be backed up before upgrades.
- The upstream image rewrites static assets at startup, so its root filesystem
  cannot currently be read-only. A non-root UID, dropped capabilities,
  `no-new-privileges`, resource limits, an isolated network, and a digest pin
  reduce the residual risk.

## Deliberately excluded

Nextcloud is not connected to Open WebUI or `home-agent` in this milestone.
Future search, document creation, and file updates require a separate,
least-privilege Nextcloud tool service with explicit path/action policies,
confirmation for writes, audit logs, and dedicated credentials.

## Legend

- Solid blue node/relationship: currently deployed component or route
- Dashed beige relationship: prepared but not deployed
- Grey node: external software system
