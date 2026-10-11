# Home Infrastructure — System Context Diagram

- **C4 type:** System context
- **Scope:** Home Infrastructure software system
- **Status:** Current as of 2026-10-11

```mermaid
flowchart LR
    julian["Julian<br/>[Person]<br/>Uses the services, operates and changes the infrastructure"]
    clients["Remote clients<br/>[External Software System]<br/>Browsers, Nextcloud/mail/Paperless apps, WebDAV"]
    mailers["Other mail servers<br/>[External Software System]<br/>Deliver mail to jkandler.de"]

    home["Home Infrastructure<br/>[Software System]<br/>Private cloud, mail, documents, AI assistant, DNS filtering, monitoring"]

    github["GitHub<br/>[External Software System]<br/>Repositories, Actions, GHCR images"]
    aws["AWS<br/>[External Software System]<br/>Route 53, Secrets Manager, SES, S3 state, OIDC/IAM, health check, budget, website"]
    anthropic["Anthropic API<br/>[External Software System]<br/>Claude chat and tool use"]
    openai["OpenAI API<br/>[External Software System]<br/>Whisper speech-to-text"]
    letsencrypt["Let's Encrypt<br/>[External Software System]<br/>TLS certificates"]
    fritz["FRITZ!Box<br/>[External Software System]<br/>Router: NAT, port forwards, WireGuard VPN, DynDNS updates"]

    julian -->|"Uses services over HTTPS, IMAP/SMTP"| home
    julian -->|"Changes code via PRs; runs Ansible and local Terraform"| github
    clients -->|"HTTPS, IMAPS, SMTPS via port forwards"| fritz
    mailers -->|"SMTP port 25 via port forward"| fritz
    fritz -->|"Forwards 80/443/25/465/993"| home
    fritz -->|"Reports the public IPv4"| aws
    home -->|"Self-hosted runners execute CI; pulls images"| github
    github -->|"OIDC-authenticated plan/apply"| aws
    home -->|"Reads secrets; relays outbound mail; DNS-01 challenges"| aws
    home -->|"Prompts and tool results over HTTPS"| anthropic
    home -->|"Audio for transcription over HTTPS"| openai
    home -->|"ACME orders"| letsencrypt
    aws -->|"Alerts when the homeserver is unreachable"| julian

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef system fill:#1168bd,color:#fff,stroke:#0b4884
    classDef external fill:#666,color:#fff,stroke:#333
    class julian person
    class home system
    class clients,mailers,github,aws,anthropic,openai,letsencrypt,fritz external
```

## Element responsibilities

| Element | C4 type | Responsibility |
| --- | --- | --- |
| Julian | Person | Sole user and operator; merges every PR |
| Home Infrastructure | Software system | Everything running on the homeserver and its k3s VMs |
| Remote clients | External software system | Reach the public hostnames listed in `current-state.md` |
| Other mail servers | External software system | Inbound SMTP to Stalwart |
| GitHub | External software system | Source of truth for all code; CI orchestration; private GHCR images |
| AWS | External software system | DNS, secrets, outbound mail relay, Terraform state, CI identities, outside-in health check, cost alerts, the static website |
| Anthropic API | External software system | Model behind `home_agent` |
| OpenAI API | External software system | Whisper behind the transcription relay |
| Let's Encrypt | External software system | Certificates for Traefik (DNS-01) and Stalwart (HTTP-01) |
| FRITZ!Box | External software system | Home router; configured by hand, not in any repo |

## Legend

- Dark blue: person
- Blue: software system in scope
- Grey: external software system
- Arrow label: intent of the directed relationship
