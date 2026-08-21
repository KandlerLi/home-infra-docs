# ADR 0013: Drop the Nextcloud write confirmation step

- Status: Accepted
- Date: 2026-08-22
- Deployment status: Implemented, not yet deployed

## Context

ADR 0011 gave the agent Nextcloud write access (create/update/delete/move)
through a two-phase `propose_nextcloud_write`/`confirm_nextcloud_write`
flow: the model proposed a write, showed the user a confirmation code, and
only executed after the user's own next chat message contained that code
verbatim (verified server-side in `home-agent`, not trusted from the
model). That design was the user's own explicit choice at the time, made
via a direct question weighing it against an autonomous alternative.

After deploying and actually using it, the user reported the confirmation
step itself as friction ("that explicite asking where to write it is
annoying") and asked for it to be removed.

## Decision

Collapse the two tools into one: `write_nextcloud_file` validates and
executes a write in a single call, immediately. There is no proposal
stage, no confirmation code, and no round-trip waiting for the user's next
message.

**What stays from ADR 0011, unchanged** — these were never about
confirmation, they're independent safety properties tied to the write
itself:

- Scope: still the same one bounded `Shared/AI Workspace` folder.
- The extension/size allowlist (`READABLE_EXTENSIONS`, `MAX_WRITE_BYTES`)
  — writes stay symmetric with what the agent can already read.
- Conflict protection: conditional WebDAV headers
  (`If-Match`/`If-None-Match`/`Overwrite`) captured from the freshly
  fetched current state immediately before the write, so a file that
  changed concurrently still fails loudly rather than being silently
  overwritten.
- The empty-folder-only delete guard — recursive deletion of a whole
  subtree in one call remains out of scope.
- Content-free audit logging of every write attempt and outcome.

**What's gone**: the `PendingWriteStore` staging layer, the confirmation
code, and the raw-user-message verification in `agent.py` that existed
solely to gate the confirm step. `SYSTEM_INSTRUCTIONS` no longer describes
a propose/confirm flow — the model is told writes happen immediately and
to report what it did after the fact.

## Consequences

- The agent can now modify files in the bounded folder without a human
  reviewing each change first. The blast radius of a single action is
  still limited by everything ADR 0011 established (folder scope,
  extension/size limits, conflict protection, no recursive delete) — what
  changed is *when* a human finds out, not *what* the agent is capable of
  touching. A mistaken or misinterpreted request now takes effect before
  anyone reviews it, recoverable only via Nextcloud's own version history
  for updates, or not at all for delete/move.
- This does not change the reasoning that led to ADR 0012's audio-relay
  design or ADR 0011's credential/scope boundaries — those are independent
  of whether a write requires confirmation.
- Net simplification: `nextcloud_tools_service.py` lost the
  `PendingWrite`/`PendingWriteStore` classes and roughly half of its write
  handling code; `agent.py` lost the confirmation-gate dispatch logic
  entirely. Removing code paths that no longer had a reason to exist,
  rather than leaving them dormant, was a deliberate choice consistent
  with this repo's general practice.
