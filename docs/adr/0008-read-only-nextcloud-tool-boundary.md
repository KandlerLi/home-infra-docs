# ADR 0008: Put Nextcloud access behind a dedicated bounded service

- Status: Accepted
- Date: 2026-08-19
- Deployment status: Private service/share deployed; diagnostics pending

## Context

The user wants the homeserver agent to locate photos and documents and later
create or update files. Open WebUI is an intentionally untrusted frontend, and
`home-agent` is an unprivileged model-facing container. Giving either component
a Nextcloud password, direct data mount, unrestricted WebDAV client, or bulk
index would cross the established security and privacy boundary.

## Decision

Introduce Nextcloud capabilities through a dedicated host-side
`nextcloud-tools` systemd service. It holds a device/app password for a
non-admin Nextcloud account that can see only one explicitly shared folder.
The service connects to AIO Apache exclusively on `127.0.0.1:11000` and exposes
fixed operations to `home-agent` over a group-protected Unix socket.

The first milestone implements only:

- direct child listing below the approved root
- bounded case-insensitive filename/path search
- UTF-8 reads for approved text extensions up to 256 KiB

Every caller path is relative and traversal-checked. Search is bounded to 500
entries, depth four, and 50 results. Returned metadata is allowlisted. The
service implements no WebDAV write method, and the agent receives neither the
app password nor direct Nextcloud network access.

Use a guarded one-time Ansible playbook to create the dedicated account with an
unrevealed generated login password and create a limited noninteractive app
password. Keep the token only in Ansible's non-cached in-memory result during
the apply, then install it directly as the service-owned `0400` file with every
secret-bearing task under `no_log`. Do not store it in SOPS or Docker.

The human owner must still share `AI Workspace` read-only after account
creation. Automating that share would require giving Ansible a credential for
the account that owns the personal files. Do not deploy without explicit
production approval. Add writes only in a later ADR with per-operation
confirmation, conflict protection, and safe auditing.

The first production bootstrap created the dedicated account and a managed
token, then stopped before secret installation because a backslash validation
expression was incorrectly YAML-escaped. Revision `e2ef617` fixes the
expression and adds compile coverage for every assertion. Guarded recovery
subsequently validated and installed a replacement, deployed the private
service/socket, and revoked the unusable token. The owner share is now mounted,
but the first bounded listing returned a generic 503. Privacy-safe diagnostics
log only upstream HTTP status or exception class; they never log paths,
filenames, content, or credentials. Their reapply revealed that shutdown-time
socket cleanup could unlink a replacement listener during restart. Systemd
runtime-directory replacement also leaves an existing container bind mount on
a stale inode. Revision `cb6a3d2` preserves the runtime directory across
restarts and forces `home-agent` recreation during dedicated deploy, rotation,
and rollback plays. Because journald returned no service records, revision
`8ceb715` adds only an allowlisted `upstream_http_NNN`, transport-class, or
generic invalid-response reason to a 503 socket response. Unstructured details
map to the generic category. Functional validation remains pending.

## Consequences

- A user-requested filename, metadata record, or text document may be sent to
  the configured cloud model; there is no background or bulk indexing.
- Photo files can be found by name, path, date, MIME type, and preview metadata,
  but image content is not analysed.
- PDF and Office contents remain unreadable in the first milestone.
- Compromise of the service credential is bounded by the dedicated account and
  read-only share; the token remains revocable independently.
- Deployment and rollback recreate `home-agent` to attach or detach only the
  Unix socket. Rollback stops the credential-bearing service without deleting
  its configuration.
- A separately guarded rotation play validates a replacement token over
  loopback WebDAV before installation and revokes only superseded tokens with
  the managed name.
