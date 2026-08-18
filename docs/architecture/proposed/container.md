# Private Homeserver Agent — Container Diagram

- **C4 type:** Container
- **Scope:** Private Homeserver Agent software system
- **Status:** Proposed; not deployed

```mermaid
flowchart LR
    person["Julian<br/>[Person]"]
    cloud["OpenAI Responses API<br/>[External Software System]"]
    docker["Docker Engine<br/>[External Software System]"]
    systemd["systemd<br/>[External Software System]"]
    procfs["Linux procfs and Filesystems<br/>[External Software System]"]

    subgraph agent_system["Private Homeserver Agent [Software System]"]
        api["home-agent API<br/>[Container: Python in Docker]<br/>Accepts requests and orchestrates model tools"]
        tools["home-tools<br/>[Container: Python systemd service]<br/>Runs fixed read-only host collectors"]
        key["OpenAI API Key File<br/>[Container: Secret data store]<br/>SOPS-derived runtime credential"]
    end

    person -.->|"Would call JSON API on authenticated HTTPS or local loopback"| api
    api -.->|"Would call Responses API with declared functions over HTTPS"| cloud
    api -.->|"Would call fixed GET endpoints over a Unix socket"| tools
    key -.->|"Would provide credential through read-only file mount"| api
    tools -.->|"Would read sanitised container-list data through Docker API"| docker
    tools -.->|"Would list failed units with fixed systemctl arguments"| systemd
    tools -.->|"Would read load, memory, uptime, and approved mounts"| procfs

    classDef person fill:#08427b,color:#fff,stroke:#052e56,stroke-dasharray:5 5
    classDef external fill:#666,color:#fff,stroke:#333,stroke-dasharray:5 5
    classDef container fill:#85bbf0,color:#111,stroke:#0b4884,stroke-dasharray:5 5
    classDef datastore fill:#2e8b57,color:#fff,stroke:#1d5c3a,stroke-dasharray:5 5
    class person person
    class cloud,docker,systemd,procfs external
    class api,tools container
    class key datastore
```

## Container catalogue

| Container | Technology | Responsibility |
| --- | --- | --- |
| `home-agent` API | Python 3.13, OpenAI Python SDK, Docker | Validate API requests, perform bounded tool-calling loop, return natural-language response |
| `home-tools` | Python 3 standard library, systemd | Expose fixed read-only structured host checks over a Unix socket |
| API key file | Root-controlled host file and read-only bind mount | Avoid Docker environment-variable exposure of the API key |

## Security properties

- The model chooses from declared tools but cannot define commands or paths.
- The model-facing container does not receive `/var/run/docker.sock`.
- `home-tools` extracts only container name, image, state, and status.
- Tool calls accept no arguments in the first milestone.
- Requests and tool rounds have fixed size/count limits.

## Legend

- Blue: person
- Grey: external software system
- Light blue: proposed C4 application container
- Green: proposed secret data store
- Dashed styling: proposed/not deployed
