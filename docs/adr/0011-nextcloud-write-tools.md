# ADR 0011: Give the home agent Nextcloud write access via a server-verified propose/confirm flow

- Status: Accepted
- Date: 2026-08-21
- Deployment status: Implemented, not yet deployed

## Context

ADR 0008 put Nextcloud access behind a dedicated `nextcloud-tools` service
and deliberately implemented only reads: "Add writes only in a later ADR
with per-operation confirmation, conflict protection, and safe auditing."
The user asked for exactly that: the agent should be able to create,
update, delete, and move files/folders in the same bounded `Shared/AI
Workspace` folder it can already read, with every write requiring explicit
human confirmation before it happens.

The user chose full CRUD (not a narrower create-only or create+update
subset) and confirmed every write must require explicit confirmation, not
run autonomously.

The key architectural constraint, researched before designing this:
**Open WebUI has no real, execution-blocking approval UI to build on.**
Its Tools/Functions/Actions subsystem is disabled entirely in this
deployment (`ENABLE_PLUGINS: "False"`, every `TOOLS_ACCESS` permission
`"False"`) — `home-agent` is exposed to it purely as an OpenAI-compatible
chat backend (`OPENAI_API_BASE_URL: http://home-agent:8000/v1`), not as a
Tool Server. Even where Open WebUI's Tools subsystem is enabled, upstream
issue [open-webui/open-webui#26073](https://github.com/open-webui/open-webui/issues/26073)
(filed June 2026, still open) confirms it renders any approval-required
text as a plain chat message with no actionable approval UI, button, or
modal — nothing in its execution pipeline can pause a tool call pending a
UI click. "Confirmation" therefore has to be a text-based two-phase flow,
engineered so the actual enforcement lives in trusted server code rather
than in model behavior, since a prompt-injected model must not be able to
talk its way past it.

## Decision

Two new tools, not one per CRUD verb, added to the existing
`nextcloud-tools` Unix-socket service and its `home-agent` client
(`ansible/roles/nextcloud_tools/files/nextcloud_tools_service.py`,
`ansible/roles/home_agent/files/agent/home_agent/nextcloud_tools.py`):

- `propose_nextcloud_write(operation, path, content, destination_path)` —
  `operation` is `create`, `update`, `delete`, or `move`. Validates
  path/extension/size exactly like the existing reads (reusing
  `normalize_relative_path` and the same `READABLE_EXTENSIONS` allowlist —
  writes stay symmetric with what the agent can already read, no new
  binary-upload capability). Fetches current server-side state (ETag, and
  for `update` the current content, to build a unified diff). Generates a
  short, human-typeable confirmation code, stores a pending-write record
  keyed by it in a bounded, expiring, in-memory store, and returns the
  code plus a human-readable summary. **Nothing is written yet.**
- `confirm_nextcloud_write(code)` — looks up the pending record and, if
  found, unexpired, and its ETag precondition still holds, performs the
  actual WebDAV write and consumes the record. Otherwise returns a clear
  `expired`/`conflict`/`not_found` error telling the model to re-propose.

**The real security boundary is in `agent.py`, not the socket service.**
When the model calls `confirm_nextcloud_write`,
`OpenAIResponsesProvider.respond()`'s dispatch loop independently verifies
— before ever forwarding the call to the socket — that the confirmation
code appears verbatim in the text of the most recent `role: "user"` entry
in the conversation. That conversation is the caller-supplied history Open
WebUI resends every turn; only genuinely human-authored turns carry that
role, so this cannot be spoofed by model output. If the code isn't there,
the call is refused locally with `confirmation_not_verified` and the
socket is never touched. A prompt-injected model can call the tool early,
but it cannot fabricate a human having typed the code.

**Conflict protection**: conditional WebDAV headers captured at propose
time — `If-Match` for update/delete/move, `If-None-Match: *` for create,
`Overwrite: F` for move — so a file that changed in between fails loudly
(a clear `conflict` error) instead of being silently overwritten.

**Folder delete requires the folder to be empty first** (checked before
issuing WebDAV `DELETE`). Recursive deletion of a whole subtree in one
confirmed action is a materially larger blast radius than every other
operation and isn't needed for the stated use case (notes and generated
files); deleting contents individually stays possible.

**Safe auditing**: a content-free `LOGGER.info` line for every propose and
every confirm attempt (operation, path, outcome), matching the style of
the service's existing failure logging. `content` is never logged.

**Pending-write store**: a plain in-memory dict inside the single,
non-replicated `nextcloud-tools` systemd process — the same process model
the read tools already run in. Bounded to a small maximum, each entry
expires after ~10 minutes. A restart just drops pending proposals; the
model re-proposes.

**One non-code prerequisite**: ADR 0008 recorded that the dedicated
`nextcloud-tools` account's share of `AI Workspace` was created read-only
by the human owner, and that automating the share itself was rejected (it
would require giving Ansible the owner's personal credential). That
manual step stands — before any of this can actually write anything, the
user must update that existing share's permissions in the Nextcloud UI to
grant create/change/delete. Skipping this means every confirmed write
fails at the WebDAV layer with 403, correct code notwithstanding.

## Consequences

- The agent can now genuinely modify the user's Nextcloud files within
  `Shared/AI Workspace`, not just read them — the private-data exposure
  surface from ADR 0008 (a requested filename, metadata record, or text
  document may be sent to the configured cloud model) now also includes
  content the agent itself generates being written back.
- Confirmation is enforced by `home-agent`'s own code checking the raw
  human message, not by any UI affordance or by trusting the model's
  claim that the user approved something — this is deliberately weaker
  than a real approval button (no rich diff rendering, the human must
  actually read the proposal) but is a genuine server-side boundary, not
  security theater.
- Two pre-existing hardcoded request-size caps (8 KiB on the service's
  `do_POST`, 4 KiB on the client's `call()`) were sized only for the small
  fixed-shape arguments the three read tools ever needed. Both were raised
  specifically for `propose_nextcloud_write`, which can now carry up to
  256 KiB of file content — found and fixed during implementation, not
  discovered live.
- `delete` is scoped to files and empty folders only; recursive folder
  deletion is out of scope for this milestone and would need its own
  design (a bulk preview, likely a higher confirmation bar) if ever added.
- Compromise of the service credential is bounded exactly as before (ADR
  0008's dedicated account, single shared folder); write access widens
  what that compromise could do within that same boundary, from read-only
  disclosure to disclosure plus tampering, bounded by conflict protection
  and confirmation.
