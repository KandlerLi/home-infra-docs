# ADR 0023: Paperless-ngx on k3s as the personal document archive

- Status: Proposed
- Date: 2026-09-25
- Deployment status: not built

## Context

Julian wants to digitize his paper documents (letters, contracts, tax
and payslip records) and keep them on his own server, searchable, rather
than as loose scans. Day-to-day capture is the phone. An existing HP
OfficeJet 4630 (35-sheet ADF, simplex scan only) is available as an
option for a one-off backlog of older paper.

Options considered:

- **Plain Nextcloud folders** (optionally with a Nextcloud OCR app):
  nothing new to run, but no document metadata (correspondent, type,
  date), weak full-text search, and no automatic classification.
- **Paperless-ngx**: the de-facto standard self-hosted document
  management system. OCR (German and English via Tesseract), full-text
  index, learned auto-tagging, a REST API, several mobile apps. Keeps
  every original as a normal file on disk, so leaving it later means
  copying files, not migrating out of a proprietary store.
- **Papra, Docspell, Mayan EDMS**: Papra is lighter but young; Docspell
  and Mayan are heavier to run. None offers a clear advantage over
  Paperless-ngx for a single user.

## Decision

Run **Paperless-ngx on the k3s cluster**, managed in `infra/k3s-apps`
like every other workload, reachable at `docs.jkandler.de`.

**Storage.** Documents are live data, so they go on the black HDD (the
workspace's convention: black for live data, red for backups):

- `media` (originals + OCR'd PDF/A archive versions + thumbnails): NFS
  from `/mnt/black-hdd/paperless-media`, a static PV in
  `bootstrap/k3s-bootstrap` like Deluge's.
- Database: **PostgreSQL**, on a `local-path` PVC via the existing
  `pvc_local_path` module (same pattern as `blocky-postgres`), not
  SQLite. Paperless-ngx's own docs advise against SQLite on network
  storage, and the node-local disk is the only non-NFS option here.
- Paperless `data` (search index, classifier model): `local-path` PVC.
  Rebuildable from `media` + the database (`document_index reindex`,
  `document_create_classifier`), so it needs no backup of its own.
- Redis (task broker): no persistence needed.

**Capture.** Two paths, both ending in an upload through the web UI or
API. Neither needs a consume folder on the server, and the scanner never
talks to the server directly.

- *Phone, day to day:* upload from the iPhone through a Paperless
  mobile app (e.g. Swift Paperless), photo → upload → Paperless does
  OCR and classification. Which app is used, and whether its built-in
  document scanner gives good enough OCR input, is to be confirmed
  during rollout, not assumed here.
- *Scanner, backlog batches (optional):* the OfficeJet 4630 scans a
  stack through its ADF on the laptop (SANE, via `sane-airscan` or
  HPLIP -- which one works is still to be confirmed) into one PDF. A
  PATCH-T separator sheet goes behind each document;
  `PAPERLESS_CONSUMER_ENABLE_BARCODES` splits the upload at those
  sheets and drops them, so one scan becomes several documents. Since
  the 4630 cannot scan duplex, double-sided stacks are scanned twice
  (fronts, then the flipped stack) and interleaved on the laptop
  before upload, e.g. `qpdf --collate --empty --pages front.pdf
  back.pdf z-1 -- out.pdf`. Paperless's own double-sided collation is
  not used: it only works through a consume subdirectory, which this
  design avoids. Wrong splits are fixed afterwards with the web UI's
  split/merge.

No mail ingestion for now; it can be added later without changing this
design.

**Access and authentication.** The phone must upload from outside the
LAN, so the host is public behind the k3s Traefik ingress, like
`mail.jkandler.de`. Two paths, deliberately different:

- Web UI: Authelia via OIDC (Paperless-ngx's `django-allauth`
  OpenID Connect provider, a new `paperless` client in
  `modules/authelia`), with the regular username/password form disabled
  (`PAPERLESS_DISABLE_REGULAR_LOGIN`).
- API (mobile app): a Paperless API token for Julian's user. Not put
  behind Authelia forward-auth, since the app cannot complete an
  interactive SSO flow; Paperless enforces authentication on every API
  call itself.

**Backup.** A nightly CronJob runs Paperless-ngx's own
`document_exporter` (originals, archive files and a metadata manifest,
restorable with `document_importer` into any later Paperless version)
into `/mnt/red-hdd/paperless-export` over NFS, the same pattern as
`authelia-backup`. This covers both the database and the originals in
one version-independent format. Accidental deletions are covered for 30
days by Paperless's own trash, not by backup history. Still on the same
physical server: off-site remains the separate `BACKLOG.md` item, and
it becomes more pressing once tax and contract documents live here.

**Explicitly out of scope:** giving `home_agent` access to Paperless.
If wanted later, it needs its own decision against
`security-invariants.md` and ADR 0021 (a dedicated read-only Paperless
user and token), not an extension of this one.

## Consequences

- One more public hostname holding the most sensitive personal data on
  the server. The API-token path is outside Authelia, so Paperless's
  own auth is the only gate there; keep the image current (trivy image
  scanning via `gha-common`) and the token in `pass`, never in git.
- Paperless-ngx upgrades can include database migrations; the nightly
  export is the rollback path.
- Four new pods (Paperless, PostgreSQL, Redis, export CronJob) on a
  cluster that already runs close to its memory budget. Check node
  headroom before applying; OCR is CPU-heavy only while a document is
  being processed.

## Rollout

One repo at a time, each through the usual PR flow:

1. `infra/home-infra`: create `/mnt/black-hdd/paperless-media` and
   `/mnt/red-hdd/paperless-export` with the right ownership, add both to
   `nfs_server_exports`.
2. `aws/secrets-manager`: a `k3s-apps/paperless` container (Postgres
   password, `PAPERLESS_SECRET_KEY`, OIDC client secret); the Authelia
   side's client secret hash in `home-infra/authelia`. Values set by
   Julian.
3. `bootstrap/k3s-bootstrap`: the two static NFS PVs.
4. `aws/dyndns`: the `docs.jkandler.de` record.
5. `infra/k3s-apps`: `modules/paperless` (Deployment, Postgres, Redis,
   PVCs, Service, ingress route, export CronJob), the Authelia OIDC
   client, a landing-page entry, and a blackbox probe.
6. Confirm live: SSO web login works and the password form is gone; an
   iPhone photo uploads through the app and comes back OCR'd and
   searchable; a scanner PDF with two separator sheets comes back as
   three documents; the nightly export lands on the red HDD; a test restore
   of that export into a throwaway instance works.

## Update 2026-09-27: Outlook mail ingestion via OAuth2

Julian wants his existing Outlook.com mail attachments imported into
Paperless, not only new ones. This replaces "No mail ingestion for now"
above for that one account.

Outlook.com no longer accepts a password or app password over IMAP, and
the account has 2FA, so the plain IMAP form is not an option. Paperless's
built-in OAuth2 mail support is used instead:

- A Microsoft app registration (personal Microsoft accounts), redirect
  URI `https://docs.jkandler.de/api/oauth/callback/`. It lives in
  Microsoft's portal, outside Terraform: a manual, documented one-off
  (runbook `k3s-apps/paperless` in `aws/secrets-manager`), like the
  Stalwart UI settings.
- Its client ID and secret go into the existing `k3s-apps/paperless`
  Secrets Manager group and reach Paperless as
  `PAPERLESS_OUTLOOK_OAUTH_CLIENT_ID`/`_SECRET`. The secret has a
  Microsoft-enforced expiry and is tracked in the rotation schedule.
- The account is then connected in Paperless's mail settings with an
  interactive Microsoft login (including 2FA).

Considered and not chosen: forwarding selected mail to a dedicated
`scan@jkandler.de` Stalwart mailbox. It limits what Paperless can see to
what is forwarded, but cannot reach mail that already exists.

Consequences:

- Paperless holds a refresh token with `IMAP.AccessAsUser.All`, i.e.
  read access to the whole Outlook mailbox, in its database. The
  database is covered by the nightly export; the token is not (the
  exporter omits it), so a restore needs the account reconnected.
- Mail rules decide what gets consumed. For the historic import: PDF
  attachments only, a maximum age long enough to cover the mailbox, and
  an action that does not depend on read state (with "mark as read",
  already-read mail is skipped). Paperless also tracks processed
  message UIDs itself.
- A large one-off import runs through the single task worker; expect
  hours, not minutes, and duplicates are consumed rather than rejected
  unless `PAPERLESS_CONSUMER_DELETE_DUPLICATES` is set.
