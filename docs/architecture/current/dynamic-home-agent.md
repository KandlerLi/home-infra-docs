# home_agent — Dynamic Diagram: chat request with a tool call

- **C4 type:** Dynamic
- **Scope:** One chat message in Open WebUI that makes the model read a
  Nextcloud file
- **Status:** Current as of 2026-10-11

```mermaid
sequenceDiagram
    autonumber
    actor J as Julian (browser)
    participant T as Traefik
    participant O as Open WebUI
    participant A as home_agent
    participant C as Anthropic API
    participant S as nextcloud_tools sidecar
    participant N as Nextcloud AIO

    J->>T: HTTPS ai.jkandler.de (OIDC session via Authelia)
    T->>O: Route (open-webui chain, no forward-auth)
    O->>A: POST /v1/chat/completions, model "home-agent"
    A->>C: Messages API, pinned model, tool definitions
    C-->>A: tool_use read_nextcloud_text_file(path)
    A->>S: Unix socket /v1/read
    S->>S: Path policy, type and size checks
    S->>N: WebDAV GET as home-agent
    N-->>S: File content
    S-->>A: Bounded result
    A->>C: tool_result
    C-->>A: Final answer
    A-->>O: OpenAI-compatible response
    O-->>J: Rendered reply
```

Steps 4–11 repeat for each tool round, at most 6 (`DEFAULT_MAX_TOOL_ROUNDS`).
Health questions use `home-tools` over `192.168.101.1:8095` instead of the
sidecar.
