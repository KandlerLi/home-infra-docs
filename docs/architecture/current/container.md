# Home Infrastructure — Container Diagram

- **C4 type:** Container
- **Scope:** Home Infrastructure software system
- **Status:** Current/configured as of 2026-08-18

In C4 terminology, a container is an application or data store. Docker and VM
placement is shown separately in the deployment diagram.

```mermaid
flowchart LR
    person["Julian<br/>[Person]<br/>Administrator and service user"]
    github["GitHub<br/>[External Software System]"]

    subgraph home["Home Infrastructure [Software System]"]
        automation["Infrastructure Automation<br/>[Container: Ansible/YAML]<br/>Defines and applies reproducible host state"]
        aio_master["Nextcloud AIO Control Plane<br/>[Container: Docker application]<br/>Manages the Nextcloud application suite"]
        nextcloud["Nextcloud Application Suite<br/>[Container: Docker applications]<br/>Files, WebDAV, collaboration, office, and talk"]
        postgres["Nextcloud Database<br/>[Container: PostgreSQL data store]<br/>Stores application metadata"]
        redis["Nextcloud Cache<br/>[Container: Redis data store]<br/>Provides cache and locking"]
        data["Nextcloud File Data<br/>[Container: ext4 data store]<br/>Stores user file content"]
        runner["GitHub Actions Runner<br/>[Container: Debian VM application]<br/>Executes trusted CI jobs in an isolated guest"]
        runner_disk["Runner VM Disk<br/>[Container: qcow2 data store]<br/>Persists the runner operating system"]
    end

    person -->|"Edits and runs playbooks"| automation
    automation -->|"Configures host packages, storage, Docker, AIO, and libvirt"| aio_master
    automation -->|"Provisions and configures over libvirt and SSH"| runner
    person -->|"Uses HTTPS and WebDAV"| nextcloud
    aio_master -->|"Creates and manages sibling containers through Docker API"| nextcloud
    nextcloud -->|"Reads and writes metadata over PostgreSQL protocol"| postgres
    nextcloud -->|"Uses cache and file locking over Redis protocol"| redis
    nextcloud -->|"Reads and writes private files"| data
    github -->|"Assigns trusted Actions jobs over HTTPS"| runner
    runner -->|"Reads and writes its VM filesystem"| runner_disk

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef container fill:#438dd5,color:#fff,stroke:#0b4884
    classDef datastore fill:#2e8b57,color:#fff,stroke:#1d5c3a
    class person person
    class github external
    class automation,aio_master,nextcloud,runner container
    class postgres,redis,data,runner_disk datastore
```

## Container catalogue

| Container | Technology | Responsibility |
| --- | --- | --- |
| Infrastructure Automation | Ansible 14.3, YAML, community collections | Converges declared infrastructure state |
| Nextcloud AIO Control Plane | Nextcloud AIO master Docker image | Manages the AIO application lifecycle |
| Nextcloud Application Suite | Nextcloud, Apache, Collabora, Talk, Imaginary, Notify Push, Whiteboard | User-facing private cloud features |
| Nextcloud Database | PostgreSQL Docker container | Application metadata |
| Nextcloud Cache | Redis Docker container | Cache and locking |
| Nextcloud File Data | ext4 on `/mnt/black-hdd` | Persistent user files |
| GitHub Actions Runner | Debian 13 VM and GitHub runner application | Trusted CI job execution |
| Runner VM Disk | qcow2 on ext4 | Guest OS persistence |

## Legend

- Blue: person
- Grey: external software system
- Light blue: C4 application container
- Green: C4 data-store container
- Arrow label: protocol or intent of the relationship
