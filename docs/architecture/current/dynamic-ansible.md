# Infrastructure Apply — Dynamic Diagram

- **C4 type:** Dynamic
- **Scope:** One reviewed Ansible deployment
- **Status:** Current workflow

```mermaid
sequenceDiagram
    autonumber
    actor Julian
    participant Git as home-infra Git working tree
    participant SOPS as SOPS/GPG
    participant Ansible as Ansible controller
    participant Host as Debian homeserver
    participant Docker as Docker Engine
    participant Libvirt as libvirt/KVM
    participant GitHub as GitHub API

    Julian->>Git: Review git status and diff
    Julian->>Ansible: Run syntax check and connectivity check
    Ansible->>SOPS: Decrypt required variables in controller memory
    SOPS-->>Ansible: Return secret variables
    Julian->>Ansible: Approve and start playbook with become
    Ansible->>Host: Apply base packages and storage mount over SSH
    Ansible->>Docker: Apply Docker and Nextcloud AIO state
    Ansible->>Libvirt: Validate or provision runner network and VM
    Ansible->>GitHub: Obtain runner metadata and short-lived registration token
    Ansible->>Host: Configure runner guest over SSH
    Ansible-->>Julian: Report changed, unchanged, and failed tasks
    Julian->>Git: Review resulting configuration and operational state
```

## Flow constraints

- The human reviews changes before deployment.
- Long-lived GitHub credentials remain on the controller.
- The runner VM receives only short-lived registration credentials.
- Destructive recovery of orphaned VM or disk state is deliberately manual.
- An AI agent is not part of the current deployment sequence.

## Legend

- Actor: human operator
- Participant: C4 person, container, component, or external system instance
- Solid arrow: synchronous request/action
- Dashed arrow: result or response
