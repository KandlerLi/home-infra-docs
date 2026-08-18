# Infrastructure Automation — Component Diagram

- **C4 type:** Component
- **Scope:** Infrastructure Automation container
- **Status:** Current committed configuration as of 2026-08-18

```mermaid
flowchart LR
    operator["Julian<br/>[Person]<br/>Runs and reviews automation"]
    host["Debian Homeserver<br/>[External Software System]"]
    github["GitHub API<br/>[External Software System]"]

    subgraph automation["Infrastructure Automation [Container: Ansible]"]
        playbooks["Playbooks<br/>[Component: Ansible YAML]<br/>Orchestrate host and runner configuration"]
        inventory["Inventory and Group Variables<br/>[Component: YAML]<br/>Define hosts and non-secret settings"]
        secrets["Encrypted Variables<br/>[Component: SOPS/GPG]<br/>Provide secrets only at execution time"]
        base["Base Role<br/>[Component: Ansible role]<br/>Installs administrative base packages"]
        storage["Storage Role<br/>[Component: Ansible role]<br/>Mounts the large HDD safely"]
        docker["Docker Role<br/>[Component: Ansible role]<br/>Installs and starts Docker Engine"]
        nextcloud["Nextcloud AIO Role<br/>[Component: Ansible role]<br/>Runs the AIO master container"]
        libvirt["libvirt Host Role<br/>[Component: Ansible role]<br/>Installs KVM/QEMU and libvirt"]
        runner["GitHub Runner Role<br/>[Component: Ansible role]<br/>Provisions the VM and runner services"]
    end

    operator -->|"Invokes and reviews"| playbooks
    inventory -->|"Supplies hosts and configuration"| playbooks
    secrets -->|"Supplies decrypted runtime variables"| playbooks
    playbooks -->|"Includes"| base
    playbooks -->|"Includes"| storage
    playbooks -->|"Includes"| docker
    playbooks -->|"Includes"| nextcloud
    playbooks -->|"Includes"| libvirt
    playbooks -->|"Includes"| runner
    base -->|"Configures through SSH with become"| host
    storage -->|"Mounts storage through SSH with become"| host
    docker -->|"Configures Docker through SSH with become"| host
    nextcloud -->|"Calls the Docker API through Ansible modules"| host
    libvirt -->|"Calls libvirt through Ansible modules"| host
    runner -->|"Calls registration and runner APIs over HTTPS"| github
    runner -->|"Defines VM and configures guest over SSH"| host

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef component fill:#85bbf0,color:#111,stroke:#0b4884
    class operator person
    class host,github external
    class playbooks,inventory,secrets,base,storage,docker,nextcloud,libvirt,runner component
```

## Component responsibilities

| Component | Responsibility |
| --- | --- |
| Playbooks | Establish ordering and host scope |
| Inventory and Group Variables | Resolve `homeserver`, LAN address, Python interpreter, and repository configuration |
| Encrypted Variables | Keep GitHub tokens and future API keys encrypted in Git |
| Base Role | Base packages and administrative group membership |
| Storage Role | Safe mount-point ownership and UUID-based HDD mount |
| Docker Role | Docker APT repository, packages, and service |
| Nextcloud AIO Role | AIO volume and master-container configuration |
| libvirt Host Role | Hypervisor packages, service, and image cache |
| GitHub Runner Role | VM, private network, cloud-init, repository registration, and systemd services |

## Legend

- Blue: person
- Grey: external system
- Light blue: component inside the Ansible container
- Arrow label: intent and direction
