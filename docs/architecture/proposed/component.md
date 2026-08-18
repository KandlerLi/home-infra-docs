# home-agent API — Component Diagram

- **C4 type:** Component
- **Scope:** `home-agent` API container
- **Status:** Proposed; implemented locally but not deployed

```mermaid
flowchart LR
    caller["Local/API Client<br/>[Person or External System]"]
    cloud["OpenAI Responses API<br/>[External Software System]"]
    host_tools["home-tools Unix API<br/>[External Container]"]
    key["API Key File<br/>[External Data Store]"]

    subgraph api["home-agent API [Container: Python]"]
        http["HTTP Request Handler<br/>[Component]<br/>Validates bounded JSON requests"]
        provider["Responses Provider<br/>[Component]<br/>Runs bounded model/tool orchestration"]
        schema["Tool Registry<br/>[Component]<br/>Defines six argument-free read-only tools"]
        client["Home Tools Client<br/>[Component]<br/>Maps tool names to fixed Unix-socket paths"]
        config["Runtime Configuration<br/>[Component]<br/>Loads model, socket path, and key file"]
    end

    caller -.->|"Would POST a message to /v1/chat"| http
    http -.->|"Would request a response"| provider
    config -.->|"Would read the key once at startup"| key
    config -.->|"Would initialise"| provider
    schema -.->|"Would constrain available function calls"| provider
    provider -.->|"Would call model with tools and function outputs over HTTPS"| cloud
    provider -.->|"Would dispatch an approved tool name"| client
    client -.->|"Would GET one fixed endpoint over Unix HTTP"| host_tools
    provider -.->|"Would return final natural-language text"| http

    classDef external fill:#666,color:#fff,stroke:#333,stroke-dasharray:5 5
    classDef component fill:#85bbf0,color:#111,stroke:#0b4884,stroke-dasharray:5 5
    class caller,cloud,host_tools,key external
    class http,provider,schema,client,config component
```

## Tool registry

| Model tool | Fixed local endpoint | Data returned |
| --- | --- | --- |
| `get_system_health` | `/v1/health` | Combined status for every approved check |
| `get_cpu_and_load` | `/v1/system` | Hostname, uptime, CPU count, and load averages |
| `get_memory_usage` | `/v1/memory` | Memory and swap totals and usage |
| `get_disk_usage` | `/v1/disks` | Usage of `/` and `/mnt/black-hdd` only |
| `get_systemd_failures` | `/v1/systemd/failed` | Failed unit name, state, and bounded description |
| `get_docker_status` | `/v1/docker/containers` | Container name, image, state, and bounded status |

No tool accepts a command, path, container name, unit name, or free-form
argument.

## Legend

- Grey: person, external system, external container, or external data store
- Light blue: component inside `home-agent`
- Dashed styling: proposed/not deployed
