# ADR 0014: Nextcloud Shopping List item tools

- Status: Accepted
- Date: 2026-08-22
- Deployment status: Implemented, not yet deployed

## Context

The user installed (or is installing) Nextcloud's **Shopping List** app
(`shopping_list`, by otherworld-dev) and wants "Put milk on my shopping
list" in chat to actually add an item to that app. Without this, the
existing generic Nextcloud file-write tool (ADR 0011/0013) would take that
request literally and create/append a plain text file instead — not what
the app, or the user, expects.

This adds a second Nextcloud integration alongside the existing WebDAV
file tools: one that talks to the Shopping List app's own OCS JSON REST
API. Confirmed with the user via AskUserQuestion: the agent gets the
**full item lifecycle** — add, check off, uncheck, and remove items, not
just add — matching the full-CRUD philosophy ADR 0011 already chose for
files. Per ADR 0013's standing preference, writes execute immediately;
this was designed with that preference from the start, so there is no
propose/confirm stage to remove here.

## What the app exposes

Confirmed directly against its source
(`github.com/otherworld-dev/Shopping-List`, `shopping_list/appinfo/
routes.php`): a standard `OCSController`-based REST API under
`/ocs/v2.php/apps/shopping_list/api/v1/`, covering lists (`GET/POST/PUT/
DELETE /lists`) and items (`GET/POST /lists/{id}/items`, `PUT .../items/
{id}/check`, `DELETE .../items/{id}`), each server-side call scoped by the
app's own `ListService::assertAccess`/`assertWriteAccess` against
Nextcloud's normal share table. Auth is the same Basic Auth (username +
app password) already used for WebDAV — no new credential.

Sharing a list with another account (`ShareController::create`) is the
app's own manual in-UI action, the same shape as sharing the `AI
Workspace` WebDAV folder was in ADR 0008.

## Decision: reuse the account, resolve names server-side, no new scope config

**Same dedicated account, no new credential.** The existing `home-agent`
Nextcloud account and app password (bootstrapped in ADR 0008, unchanged
since) are reused as-is for the Shopping List API.

**No `ALLOWED_ROOT`-equivalent config.** WebDAV needs the `ALLOWED_ROOT`
restriction in `nextcloud_tools_service.py` because a Nextcloud account's
DAV namespace always exposes its whole home directory tree, so the service
narrows itself to one folder as defense in depth even though only that
folder is meant to be shared. The Shopping List OCS API has no equivalent
broader default surface: `GET /lists` only ever returns lists explicitly
shared with (or owned by) the authenticated account. There is nothing
broader to narrow away, so no new allowed-list environment variable was
added — scope is enforced entirely by what the user chooses to share,
exactly mirroring the existing folder-share trust boundary.

**Full item lifecycle, list lifecycle stays manual.** Item-level CRUD
(add/check/uncheck/remove) is in scope, mirroring file CRUD within the
shared root. Creating, renaming, or deleting an entire *list* is out of
scope — the user creates and shares lists by hand in the app, the same way
they create and share the `AI Workspace` folder by hand. This mirrors the
file tools' own boundary: the agent manages things *within* an
already-shared container, not the container itself.

**Name resolution happens server-side, in `nextcloud_tools_service.py`.**
The model only ever knows names ("milk", "my shopping list"), never
internal list/item IDs, so `resolve_shopping_list()` and
`resolve_shopping_list_item()` resolve names to IDs before acting:

- List: an explicit `list` argument matches case-insensitively; if
  omitted, the sole shared list is used automatically, and it's an error
  (naming the available titles) if zero or more than one are shared.
- Item (for check/uncheck/remove only — `add` never needs to resolve): a
  case-insensitive exact match wins outright; failing that, a unique
  case-insensitive substring match is used; zero or multiple matches is an
  error listing the current item names, bounded by the same `MAX_RESULTS`
  the WebDAV `search()` tool already uses, so the model can retry with the
  right spelling rather than guessing blind.

**Assumption flagged for live verification** (same discipline as the
`occ files_external` `mount_id` field and the WebDAV `If-None-Match`
behavior in earlier sessions): that Nextcloud's OCS framework accepts a
JSON request body for POST/PUT parameters the same way it accepts
form-encoded ones. Standard, documented OCP behavior, but confirmed live
during rollout, not assumed.

**Three tools, not one per verb**, mirroring the existing
list/search/read + one unified write shape already used for files:
`list_shopping_lists` (read), `list_shopping_list_items` (read),
`update_shopping_list` (operation ∈ `add|check|uncheck|remove`, write,
immediate — no confirmation stage).

**Logging is not content-free the way file writes are.** File writes log
only the `path`, never file bytes, because file content can be arbitrary
and sensitive. A shopping-list item's *name* is the entire payload of an
add/check/remove — there is no separate "content" to protect, and it's
directly analogous to the `path` the file tools already log. So
`nextcloud_tools_service.py` logs operation, resolved list title, and item
name on every attempt/outcome; this is not a relaxation of the file-write
logging stance, just the equivalent level of detail applied to a different
kind of write.

## Consequences

- The agent can add, check off, uncheck, and remove items on any Shopping
  List list the user explicitly shares with the `home-agent` account,
  immediately, with no per-write human review — consistent with ADR 0013's
  standing preference, applied here from first implementation rather than
  as a later removal.
- No new Nextcloud credential, systemd environment variable, or Ansible
  role/playbook was needed — this is additive at the Python service/client
  layer only, deployed through the same `nextcloud-tools.yml`/
  `home-agent.yml` playbooks as ADR 0011.
- The blast radius of a mistaken or ambiguous request is bounded by name
  resolution refusing to guess (ambiguous or unknown names are refused
  with a listing, not silently acted on) and by the list-creation/deletion
  boundary staying manual, but a correctly-resolved add/check/remove takes
  effect immediately and is only recoverable by asking the agent (or the
  user) to reverse it by hand.
