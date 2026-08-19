# C4 Architecture Documentation

The architecture follows the [C4 model](https://c4model.com/), using
hierarchical views rather than one all-purpose diagram.

## Views

### Current deployment

1. [System context](current/system-context.md) — people and external systems
2. [Container view](current/container.md) — major applications and data stores
3. [Infrastructure automation components](current/component-home-infra.md) —
   internals of the Ansible automation container
4. [Home-agent components](current/component-home-agent.md) — bounded request,
   model, and read-only tool orchestration
5. [Deployment view](current/deployment.md) — physical host, Docker, VM, DNS,
   and network placement
6. [Ansible deployment flow](current/dynamic-ansible.md) — runtime sequence for
   an infrastructure apply
7. [Agent health-check flow](current/dynamic-home-agent.md) — runtime sequence
   for a read-only health request
8. [AWS account-bootstrap control plane](current/account-bootstrap.md) —
   local-only identity ownership and GitHub OIDC trust path

### Retained deployment views

1. [Publication index](proposed/README.md)
2. [Deployed routing view](proposed/deployment.md)
3. [Open WebUI rollout](proposed/open-webui.md) — retained deployed rollout view

## Notation

Mermaid flowcharts are used as a portable notation. Each node includes its C4
element type in brackets.

| Visual convention | Meaning |
| --- | --- |
| Blue node | Person |
| Dark blue node | Software system in scope |
| Grey node | External software system |
| Light blue node | C4 container or component |
| Green node | Data store |
| Beige node | Deployment or infrastructure node |
| Dashed border or relationship | Proposed/not deployed |

Colours are supplementary; every node also has a textual type.

## Scope choices

The system in scope is **Home Infrastructure**: the applications, automation,
and data stores operated on or for the homeserver. AWS Route 53, the dynamic
DNS provider, GitHub, and the cloud model provider are external systems.

The C4 code level is intentionally omitted. The source code and Ansible task
files are small enough to navigate directly, while system, container,
component, deployment, and dynamic views provide the useful architectural
abstractions.

## Diagram review standard

Every diagram is checked for:

- explicit title and C4 diagram type
- stated scope and status
- named and typed elements
- responsibilities and technologies where applicable
- directional, labelled relationships
- a legend

This mirrors the official C4 diagram review checklist.
