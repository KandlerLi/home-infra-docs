# Private Homeserver Agent — System Context Diagram

- **C4 type:** System context
- **Scope:** Private Homeserver Agent software system
- **Status:** Proposed; not deployed

```mermaid
flowchart LR
    person["Julian<br/>[Person]<br/>Requests health analysis and reviews results"]
    agent["Private Homeserver Agent<br/>[Software System]<br/>Interprets requests using restricted homeserver tools"]
    host["Home Infrastructure<br/>[External Software System]<br/>Provides sanitised operational state"]
    cloud["Cloud Model Provider<br/>[External Software System]<br/>Performs language-model inference"]

    person -.->|"Would submit authenticated natural-language requests"| agent
    agent -.->|"Would request only approved read-only observations"| host
    agent -.->|"Would send prompts and selected tool results over HTTPS"| cloud

    classDef person fill:#08427b,color:#fff,stroke:#052e56,stroke-dasharray:5 5
    classDef proposed fill:#1168bd,color:#fff,stroke:#0b4884,stroke-dasharray:5 5
    classDef external fill:#666,color:#fff,stroke:#333,stroke-dasharray:5 5
    class person person
    class agent proposed
    class host,cloud external
```

## Scope notes

The cloud model does not directly connect to the host. Trusted local code
decides how every tool obtains and filters information. Nextcloud document
retrieval is outside this system context and outside the first milestone.

## Legend

- Blue: person
- Dark blue: proposed software system in scope
- Grey: external software system
- Dashed border and relationship: proposed/not deployed
