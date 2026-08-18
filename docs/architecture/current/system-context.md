# Home Infrastructure — System Context Diagram

- **C4 type:** System context
- **Scope:** Home Infrastructure software system
- **Status:** Current/configured as of 2026-08-18

```mermaid
flowchart LR
    person["Julian<br/>[Person]<br/>Operates the server and uses private services"]
    home["Home Infrastructure<br/>[Software System]<br/>Provides private cloud, automation, and isolated CI"]
    github["GitHub<br/>[External Software System]<br/>Hosts source repositories and Actions jobs"]
    route53["AWS Route 53<br/>[External Software System]<br/>Hosts DNS for jkandler.de"]
    ddns["Dynamic DNS Provider<br/>[External Software System]<br/>Tracks the changing public IPv4 address"]
    internet["Remote Client Software<br/>[External Software System]<br/>Browsers and WebDAV clients reaching Nextcloud"]

    person -->|"Uses Nextcloud over HTTPS"| home
    person -->|"Configures through Git and Ansible over SSH"| home
    internet -->|"Accesses Nextcloud over HTTPS/WebDAV"| home
    home -->|"Runs trusted self-hosted Actions jobs"| github
    internet -->|"Resolves jkandler.de"| route53
    internet -->|"Resolves jkandler.dedyn.io"| ddns
    home -->|"Publishes the changing public IPv4 address"| ddns

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef system fill:#1168bd,color:#fff,stroke:#0b4884
    classDef external fill:#666,color:#fff,stroke:#333
    class person person
    class home system
    class github,route53,ddns,internet external
```

## Element responsibilities

| Element | C4 type | Responsibility |
| --- | --- | --- |
| Julian | Person | Administration and private service use |
| Home Infrastructure | Software system | Operates Nextcloud, reproducible configuration, storage, and isolated CI |
| GitHub | External software system | Source hosting, runner registration, and Actions job coordination |
| AWS Route 53 | External software system | DNS authority for `jkandler.de` |
| Dynamic DNS Provider | External software system | Dynamic record for the public IPv4 connection |
| Remote Client Software | External software system | Browsers and WebDAV clients outside the LAN |

## Legend

- Blue: person
- Dark blue: software system in scope
- Grey: external software system
- Arrow label: intent of the directed relationship

The proposed cloud AI provider is intentionally absent because no agent is
deployed.
