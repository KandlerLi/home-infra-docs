# Runbook: Update Documentation

## When to update

Update this repository whenever a change affects:

- deployed software systems, applications, or data stores
- host, container, VM, network, port, DNS, or storage inventory
- trust boundaries or secret handling
- standard operating commands
- an architecture decision
- the status of a proposed design
- repository governance, CI/CD, OIDC, IAM, or Terraform state access

## Procedure

1. Record the current date and inspect all relevant implementation and
   documentation repositories.
2. Review committed infrastructure and uncommitted changes separately.
3. Gather only the minimum read-only runtime evidence needed.
4. Update `current-state.md` and the affected detailed pages.
5. Update C4 diagrams at every affected abstraction level.
6. Add or supersede an ADR when a durable architectural choice changes.
7. Check that every diagram has a title, scope, types, labelled arrows, and
   legend.
8. Search for likely secret material before committing.
9. Validate relative Markdown links and Mermaid code blocks.
10. Update `/home/julian/projects/home-infra-ai-context` using its
    `context/synchronization.md` contract and add a dated handover for material
    work.
11. Confirm the AI context and human documentation agree on current/proposed
    status, safety rules, and the next task.
12. Commit documentation independently from infrastructure code.

## Promote proposed architecture to current

Only move an element from `architecture/proposed` to `architecture/current`
after deployment and runtime verification. Committed code alone is not proof
of deployment.
