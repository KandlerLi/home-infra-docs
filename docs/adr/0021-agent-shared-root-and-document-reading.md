# ADR 0021: Widen the agent's Nextcloud scope and let it read PDF/xlsx

- Status: Accepted
- Date: 2026-09-19
- Deployment status: root widening (`k3s-apps#64`) and document reading
  (`home-infra` PR "read PDF and xlsx", then a k3s-apps image bump) are built
  and in review; nothing is live until those are merged and applied. The
  shares themselves are created by the owner by hand.
- Amends: 0011 (write tools), 0013; lifts the "no binary extraction" line in
  `security-invariants.md` for exactly the two formats below.

## Context

Since 2026-09-18 `home_agent` runs on Claude (Anthropic Messages API) instead
of OpenAI. The user then asked for three things: access to all their folders,
write access to their second-brain vault, and support for PDFs and `.xlsx`
(the agent could only read small UTF-8 text files).

`security-invariants.md` forbids "arbitrary filesystem access" and says no
binary extraction may be added "without a separate threat review and
decision". This ADR is that review and decision.

## Decisions

**1. Scope stays a Nextcloud share boundary, not a filesystem one.**
`NEXTCLOUD_ALLOWED_ROOT` moves from `Shared/AI Workspace` to `Shared`. The
agent's non-admin `home-agent` account still sees only what the owner shares
with it; the root change grants nothing until a share exists. "All folders"
means: the owner shares the folders they want visible. The agent's credential
can neither create nor widen shares (unchanged, ADR 0011).

**2. Write access is per share, enforced by Nextcloud.** The second-brain
share is set to "can edit"; every other share is read-only. A write to a
read-only share fails at Nextcloud (403), so the sidecar needs no new
permission logic. Writes keep the existing bounds: text extensions only,
256 KiB, conditional headers, empty-folder-only delete.

**3. PDFs go to the model natively; xlsx is flattened with the stdlib.**
- New read-only tool `read_nextcloud_document` (sidecar `/v1/read_document`).
- PDF: not parsed anywhere in the cluster. Size <= 4 MiB and `%PDF-` magic are
  checked, then the bytes are sent to Anthropic as a `document` block inside
  the `tool_result`.
- xlsx: converted to tab-separated text per sheet with `zipfile` +
  streaming `ElementTree`. No third-party parser was added.
- Rejected alternative: `pypdf`/`openpyxl` in the agent container. Better text
  extraction, but it would put a parser for attacker-influenced files next to
  the Anthropic and OpenAI API keys.

## Threat review

| Threat | Mitigation |
| --- | --- |
| Zip bomb / oversized parts | Declared sizes capped per part (8 MiB) and in total (16 MiB) before reading; only the named parts (`workbook.xml`, its rels, `sharedStrings.xml`, `worksheets/*.xml`) are ever opened; encrypted parts refused; download capped at 4 MiB |
| XML entity expansion / external entities | Any part containing `<!DOCTYPE`/`<!ENTITY` is refused; UTF-16/32 parts refused; streaming parse with element clearing, so no full DOM for sheets (sidecar has 128 MiB) |
| Output blow-up | Output capped at the existing 256 KiB read limit (`truncated` flag); max 50 sheets, 1000 columns, 32k chars per cell |
| Parser bug reaching secrets | Only stdlib code runs on the file, in the sidecar. PDFs are never parsed locally |
| Prompt injection in a document driving the agent | Tool description and system prompt say document contents are untrusted and must never cause writes. Residual risk remains: a hostile PDF could still talk the model into editing the one editable share (text files, `If-Match` protected, no folder delete unless empty). Keep hostile files out of shares, and keep editable scope to second-brain only |
| Egress to a cloud model | Reads happen only when the user asks (system prompt); nothing is indexed in the background. Any file the agent opens, including personal financial notes in second-brain, is sent to Anthropic. This is now explicit, accepted by the owner, and limited to what the owner shares |
| Large PDF cost/latency | 4 MiB cap; the PDF is re-sent with the conversation for the remaining tool rounds (max 4) |

## Consequences

- The agent's response cap is raised to 6 MiB for this one tool only; every
  other tool keeps 1 MiB.
- Dates in spreadsheets appear as serial numbers, and formulas return their
  cached value (a file that was never opened in a spreadsheet app has none).
- Scanned/image-only PDFs work only as far as the model can read them.
- The sidecar source is vendored: `home-infra/nextcloud_tools/` is canonical,
  `k3s-apps/modules/home_agent/files/` is a hand-synced copy.
- Not covered: `.docx`, `.pptx`, `.xls`, images. Each needs its own decision.
