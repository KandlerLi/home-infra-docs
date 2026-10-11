# ADR 0001: Use Ansible and Git as the Infrastructure Source of Truth

- Status: Accepted
- Date: 2026-08-18

## Context

The physical homeserver must be rebuildable without relying on undocumented
manual configuration. It runs Debian services, Docker workloads, storage, and
a libvirt guest.

## Decision

Store declarative infrastructure configuration in the `home-infra` Git
repository and apply it with Ansible.

## Consequences

- Infrastructure changes can be reviewed as Git diffs.
- Roles and playbooks must remain idempotent and safe around persistent data.
- External state such as Fritzbox rules and DNS needs documentation until it
  is also managed declaratively.
- Production applies remain human-approved.
