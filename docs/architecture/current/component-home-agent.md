# home_agent — Component Diagram

- **C4 type:** Component
- **Scope:** The `home_agent` Pod (`infra/k3s-apps` `modules/home_agent`)
- **Status:** Current as of 2026-10-11

Source: `infra/home-infra` `ansible/roles/home_agent/files/agent/` (agent,
built into `ghcr.io/kandlerli/home-agent` by CI) and
`nextcloud_tools/` (sidecar, synced into `k3s-apps`).

```mermaid
flowchart LR
    owui["Open WebUI<br/>[Container]"]
    anthropic["Anthropic Messages API<br/>[External Software System]"]
    whisper["OpenAI Whisper API<br/>[External Software System]"]
    hometools["home-tools<br/>[Container: systemd on the homeserver]"]
    aio["Nextcloud AIO<br/>[Container]"]

    subgraph pod["home_agent Pod"]
        api["HTTP API<br/>[Component: api.py]<br/>/v1/chat, /v1/chat/completions, /v1/models, /v1/audio/transcriptions; body limits"]
        loop["Agent loop<br/>[Component: agent.py]<br/>Pinned model, tool-use rounds, bounded output"]
        audio["Transcription relay<br/>[Component: audio.py]<br/>25 MB cap, model pinned, no content logging"]
        htc["Home tools client<br/>[Component: home_tools.py]<br/>Fixed health collectors, no arguments"]
        nctc["Nextcloud tools client<br/>[Component: nextcloud_tools.py]<br/>list, search, read, read_document, write, shopping list"]
        sidecar["nextcloud_tools service<br/>[Component: sidecar container]<br/>Path policy, size/type allowlist, WebDAV + OCS, audit log"]
    end

    owui -->|"Chat and audio over HTTP"| api
    api --> loop
    api --> audio
    loop -->|"HTTPS"| anthropic
    audio -->|"HTTPS"| whisper
    loop --> htc
    loop --> nctc
    htc -->|"HTTP 192.168.101.1:8095"| hometools
    nctc -->|"Unix socket in a shared emptyDir"| sidecar
    sidecar -->|"WebDAV/OCS as the home-agent account"| aio

    classDef external fill:#666,color:#fff,stroke:#333
    classDef container fill:#438dd5,color:#fff,stroke:#0b4884
    classDef component fill:#85bbf0,color:#000,stroke:#5d82a8
    class anthropic,whisper external
    class owui,hometools,aio container
    class api,loop,audio,htc,nctc,sidecar component
```

## Boundaries

The agent holds the Anthropic and OpenAI keys; Open WebUI holds neither.
The sidecar holds the Nextcloud app password; the agent container does
not. Nextcloud scope is whatever Julian shares with `home-agent` under
`Shared` (ADR 0021). The full rule set is the `check-security-invariants`
skill.

## Legend

- Grey: external system
- Medium blue: container outside this Pod
- Light blue: component
