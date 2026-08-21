# Contributing

## Update workflow

1. Enumerate every project directory nested one level under
   `/home/julian/projects/docs`, `/home/julian/projects/bootstrap`, and
   `/home/julian/projects/infra`, then check Git status in each relevant Git
   repository and explicitly note any affected project without version
   control.
2. Distinguish committed configuration, uncommitted work, and deployed state.
3. Gather live state with read-only commands only.
4. Update the relevant inventory or runbook page.
5. Update affected C4 diagrams and architecture decision records.
6. Review and update `/home/julian/projects/docs/home-infra-ai-context` so another
   model/chat receives the same current state, decisions, safety rules, and
   continuation point.
7. Add or amend a dated AI handover for material work; if no AI-context content
   change is needed, record that it was reviewed and why.
8. Verify that no secrets, Terraform state, or document contents were added.
9. Check Markdown links and Mermaid syntax before committing.

The reciprocal synchronization contract is:

```text
/home/julian/projects/docs/home-infra-ai-context/context/synchronization.md
```

It applies to relevant implementation, DNS/cloud, workflow, security,
architecture, operations, repository, and production-state changes—not only
large deployments.

## C4 diagram rules

Every diagram must state its type and scope. Every element must have a name,
type, responsibility, and technology where relevant. Every relationship must
have a directional label. Every diagram must include a legend.

The diagrams use Mermaid flowcharts as a notation for C4 concepts. The C4
model is notation- and tooling-independent, so the semantic element labels are
more important than the shapes.

## Sensitive information

Allowed examples include LAN addresses, mount points, service names, public
hostnames, and secret variable names. Do not record:

- API keys or tokens
- private keys
- decrypted SOPS content
- Docker container environments
- document names or contents from private Nextcloud collections
- transient public IP addresses
- raw Terraform state or saved plan files
