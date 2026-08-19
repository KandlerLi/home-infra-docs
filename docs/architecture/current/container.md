# Home Infrastructure — Container Diagram

- **C4 type:** Container
- **Scope:** Home Infrastructure software system
- **Status:** Current/configured as of 2026-08-19

In C4 terminology, a container is an application or data store. Docker and VM
placement is shown separately in the deployment diagram.

```mermaid
flowchart LR
    person["Julian<br/>[Person]<br/>Administrator and service user"]
    github["GitHub<br/>[External Software System]"]
    openai["OpenAI Responses API<br/>[External Software System]"]
    route53["AWS Route 53<br/>[External Software System]"]
    fritzbox["FRITZ!Box<br/>[External Software System]"]
    host["Debian, systemd, and Docker API<br/>[External Software System]"]

    subgraph home["Home Infrastructure [Software System]"]
        automation["Infrastructure Automation<br/>[Container: Ansible/YAML]<br/>Defines and applies reproducible host state"]
        dns_iac["DNS Automation<br/>[Container: Terraform]<br/>Defines Route 53 records and AWS DynDNS resources"]
        dyndns["DynDNS Update Service<br/>[Container: API Gateway and Lambda]<br/>Authenticates FRITZ!Box IPv4 updates"]
        ingress["Shared HTTPS Ingress<br/>[Container: Traefik]<br/>Terminates ACME TLS and routes public hostnames"]
        aio_master["Nextcloud AIO Control Plane<br/>[Container: Docker application]<br/>Manages the Nextcloud application suite"]
        nextcloud["Nextcloud Application Suite<br/>[Container: Docker applications]<br/>Files, WebDAV, collaboration, office, and talk"]
        postgres["Nextcloud Database<br/>[Container: PostgreSQL data store]<br/>Stores application metadata"]
        redis["Nextcloud Cache<br/>[Container: Redis data store]<br/>Provides cache and locking"]
        data["Nextcloud File Data<br/>[Container: ext4 data store]<br/>Stores user file content"]
        runner["GitHub Actions Runner<br/>[Container: Debian VM application]<br/>Executes trusted CI jobs in an isolated guest"]
        runner_disk["Runner VM Disk<br/>[Container: qcow2 data store]<br/>Persists the runner operating system"]
        agent["home-agent API<br/>[Container: Python in Docker]<br/>Orchestrates bounded model and tool calls"]
        tools["home-tools<br/>[Container: Python systemd service]<br/>Returns fixed, sanitised, read-only observations"]
        agent_key["OpenAI API Key File<br/>[Container: Secret data store]<br/>SOPS-derived runtime credential"]
    end

    person -->|"Edits and runs playbooks"| automation
    person -->|"Reviews and merges Terraform changes"| dns_iac
    dns_iac -->|"Manages stable CNAME records and updater resources"| route53
    fritzbox -->|"Submits authenticated public IPv4 updates"| dyndns
    dyndns -->|"Updates the jkandler.de A record"| route53
    automation -->|"Configures host packages, storage, Docker, AIO, and libvirt"| aio_master
    automation -->|"Provisions and configures over libvirt and SSH"| runner
    person -->|"Uses HTTPS/WebDAV and authenticated agent API"| ingress
    ingress -->|"Routes nextcloud.jkandler.de to loopback upstream"| nextcloud
    ingress -->|"Authenticates and routes ai.jkandler.de to loopback"| agent
    aio_master -->|"Creates and manages sibling containers through Docker API"| nextcloud
    nextcloud -->|"Reads and writes metadata over PostgreSQL protocol"| postgres
    nextcloud -->|"Uses cache and file locking over Redis protocol"| redis
    nextcloud -->|"Reads and writes private files"| data
    github -->|"Assigns trusted Actions jobs over HTTPS"| runner
    runner -->|"Reads and writes its VM filesystem"| runner_disk
    person -->|"Calls JSON API through host loopback"| agent
    agent -->|"Uses Responses API over HTTPS"| openai
    agent -->|"Calls fixed GET endpoints over a Unix socket"| tools
    agent_key -->|"Provides credential through a read-only file mount"| agent
    tools -->|"Reads approved host, service, filesystem, and sanitised container state"| host

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef container fill:#438dd5,color:#fff,stroke:#0b4884
    classDef datastore fill:#2e8b57,color:#fff,stroke:#1d5c3a
    class person person
    class github,openai,route53,fritzbox,host external
    class automation,dns_iac,dyndns,ingress,aio_master,nextcloud,runner,agent,tools container
    class postgres,redis,data,runner_disk,agent_key datastore
```

## Container catalogue

| Container | Technology | Responsibility |
| --- | --- | --- |
| Infrastructure Automation | Ansible 14.3, YAML, community collections | Converges declared infrastructure state |
| DNS Automation | Terraform, AWS provider, GitHub Actions OIDC | Defines CNAME records and the least-privilege DynDNS stack |
| DynDNS Update Service | API Gateway HTTP API, Python Lambda | Authenticates FRITZ!Box requests and updates the apex IPv4 record |
| Shared HTTPS Ingress | Traefik `v3.7.1`, file provider, ACME | Routes Nextcloud and the authenticated agent without Docker API access |
| Nextcloud AIO Control Plane | Nextcloud AIO master Docker image | Manages the AIO application lifecycle |
| Nextcloud Application Suite | Nextcloud, Apache, Collabora, Talk, Imaginary, Notify Push, Whiteboard | User-facing private cloud features |
| Nextcloud Database | PostgreSQL Docker container | Application metadata |
| Nextcloud Cache | Redis Docker container | Cache and locking |
| Nextcloud File Data | ext4 on `/mnt/black-hdd` | Persistent user files |
| GitHub Actions Runner | Debian 13 VM and GitHub runner application | Trusted CI job execution |
| Runner VM Disk | qcow2 on ext4 | Guest OS persistence |
| `home-agent` API | Python 3.13, OpenAI SDK, Docker | Bounded natural-language health requests and tool orchestration |
| `home-tools` | Python 3 standard library, systemd | Fixed read-only host collectors over a Unix socket |
| OpenAI API Key File | Root-controlled file, read-only bind mount | Runtime credential without Docker environment-variable exposure |

## Legend

- Blue: person
- Grey: external software system
- Light blue: C4 application container
- Green: C4 data-store container
- Arrow label: protocol or intent of the relationship
