# Home Infrastructure — System Context Diagram

- **C4 type:** System context
- **Scope:** Home Infrastructure software system
- **Status:** Current/configured as of 2026-08-19

```mermaid
flowchart LR
    person["Julian<br/>[Person]<br/>Operates the server and uses private services"]
    home["Home Infrastructure<br/>[Software System]<br/>Provides private cloud, automation, and isolated CI"]
    github["GitHub<br/>[External Software System]<br/>Hosts source repositories and Actions jobs"]
    route53["AWS Route 53<br/>[External Software System]<br/>Hosts DNS for jkandler.de"]
    aws_dyndns["AWS DynDNS Endpoint<br/>[External Software System]<br/>API Gateway, Lambda, and Secrets Manager"]
    ddns["Dynamic DNS Provider<br/>[External Software System]<br/>Tracks the changing public IPv4 address"]
    openai["OpenAI Responses API<br/>[External Software System]<br/>Performs cloud model inference"]
    internet["Remote Client Software<br/>[External Software System]<br/>Browsers and WebDAV clients reaching Nextcloud"]

    person -->|"Uses Nextcloud over HTTPS"| home
    person -->|"Configures through Git and Ansible over SSH"| home
    person -->|"Requests health analysis through authenticated HTTPS"| home
    internet -->|"Accesses Nextcloud over HTTPS/WebDAV"| home
    home -->|"Runs trusted self-hosted Actions jobs"| github
    internet -->|"Resolves jkandler.de"| route53
    internet -->|"Resolves jkandler.dedyn.io"| ddns
    home -->|"Publishes the changing public IPv4 address"| ddns
    home -->|"Submits authenticated IPv4 updates"| aws_dyndns
    aws_dyndns -->|"Updates the jkandler.de A record"| route53
    home -->|"Sends prompts and selected sanitised health data over HTTPS"| openai

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef system fill:#1168bd,color:#fff,stroke:#0b4884
    classDef external fill:#666,color:#fff,stroke:#333
    class person person
    class home system
    class github,route53,aws_dyndns,ddns,openai,internet external
```

## Element responsibilities

| Element | C4 type | Responsibility |
| --- | --- | --- |
| Julian | Person | Administration and private service use |
| Home Infrastructure | Software system | Operates Nextcloud, reproducible configuration, storage, and isolated CI |
| GitHub | External software system | Source hosting, runner registration, and Actions job coordination |
| AWS Route 53 | External software system | DNS authority for `jkandler.de` |
| AWS DynDNS Endpoint | External software system | Receives authenticated FRITZ!Box updates and changes the apex A record |
| Dynamic DNS Provider | External software system | Dynamic record for the public IPv4 connection |
| OpenAI Responses API | External software system | Cloud inference for bounded health requests and tool results |
| Remote Client Software | External software system | Browsers and WebDAV clients outside the LAN |

## Legend

- Blue: person
- Dark blue: software system in scope
- Grey: external software system
- Arrow label: intent of the directed relationship

The agent is part of Home Infrastructure at this level. Its container API is
loopback-only, while Traefik provides the public authenticated endpoint at
`ai.jkandler.de`.
