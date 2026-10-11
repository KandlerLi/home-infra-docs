# C4 Architecture Documentation

The architecture follows the [C4 model](https://c4model.com/), using
hierarchical views rather than one all-purpose diagram.

## Views

All views are current as of 2026-10-11. For the maintained prose summary see
[`overview.md`](overview.md).

1. [System context](current/system-context.md) — people and external systems
2. [Container view](current/container.md) — applications and data stores,
   grouped by where they run
3. [Deployment view](current/deployment.md) — homeserver, k3s VMs, networks,
   disks, AWS
4. [Infrastructure automation](current/component-home-infra.md) — which
   repository manages what, local vs. CI apply
5. [Repository infrastructure control plane](current/repo-infra.md) — GitHub
   settings and the AWS OIDC/IAM identities CI runs with
6. [Change rollout](current/dynamic-ansible.md) — PR to apply for Terraform,
   and an Ansible run
7. [home_agent components](current/component-home-agent.md) — API, model
   loop, tools, sidecar
8. [home_agent chat flow](current/dynamic-home-agent.md) — one chat message
   with a Nextcloud tool call

## Notation

Mermaid flowcharts are used as a portable notation. Each node includes its C4
element type in brackets.

| Visual convention | Meaning |
| --- | --- |
| Dark blue node | Person |
| Blue node | Software system in scope |
| Grey node | External software system |
| Medium blue node | C4 container |
| Light blue node | C4 component |
| Green node | Data store |
| Beige node | Deployment or infrastructure node |

Colours are supplementary; every node also has a textual type.

## Scope choices

The system in scope is **Home Infrastructure**: the applications, automation,
and data stores operated on or for the homeserver. GitHub, AWS, the model
providers, Let's Encrypt and the FRITZ!Box are external systems.

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
