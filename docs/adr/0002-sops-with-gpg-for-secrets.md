# ADR 0002: Encrypt Committed Secrets with SOPS and GPG

- Status: Accepted
- Date: 2026-08-18

## Context

Unattended services need credentials, but plaintext `.env` files and cleartext
Ansible defaults would leak secrets through Git or Docker metadata.

## Decision

Encrypt the Ansible secret-variable file with SOPS using the configured GPG
recipient. Load it through the Ansible `community.sops` vars plugin.

## Consequences

- Encrypted secret material can be versioned.
- The GPG private key remains an external recovery dependency.
- Decrypted values must use `no_log` and file-based runtime delivery where
  possible.
- Secret variable names may be documented; values may not.
