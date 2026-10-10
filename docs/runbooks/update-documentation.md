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

1. Record the current date, enumerate all top-level directories under
   `/home/julian/projects`, and inspect every relevant project—including those
   without Git.
2. Review committed infrastructure and uncommitted changes separately.
3. Gather only the minimum read-only runtime evidence needed.
4. Update `current-state.md` and the affected detailed pages.
5. Update C4 diagrams at every affected abstraction level.
6. Add or supersede an ADR when a durable architectural choice changes.
7. Check that every diagram has a title, scope, types, labelled arrows, and
   legend.
8. Search for likely secret material before committing.
9. Validate relative Markdown links and Mermaid code blocks.
10. If a security boundary changed, update the
    `check-security-invariants` skill (`/home/julian/projects/.claude/skills/`).
11. Commit documentation independently from infrastructure code.

## Promote proposed architecture to current

Only move an element from `architecture/proposed` to `architecture/current`
after deployment and runtime verification. Committed code alone is not proof
of deployment.
