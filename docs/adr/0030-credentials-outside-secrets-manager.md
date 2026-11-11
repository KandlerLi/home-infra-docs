# ADR 0030: Credentials that can't live in Secrets Manager go into `pass` or onto paper

- Status: Accepted
- Date: 2026-11-11
- Deployment status: in use; catalog and rotation entries merged in
  `secrets-manager#48`; first offline medium created and restore-tested
  on 2026-11-11

## Context

The security invariant says secret values live only in AWS Secrets
Manager. Some credentials can't:

- the IAM access keys of the scripted `*-local` identities, which are
  what reaches AWS in the first place;
- what unlocks the operator's own machines (disk encryption, login,
  sudo) and the GPG key that decrypts SOPS (ADR 0002) and `pass`;
- human logins and recovery material for accounts that sit outside AWS.

These were kept in a mix of places with no list of what exists, no
rotation rule and no tested offline copy.

## Decision

Two storage classes outside Secrets Manager, and nothing else:

- **`pass`**: the operator's GPG-encrypted password store, for every
  credential above that doesn't have to be reachable without `pass`.
- **Memory + paper**: only for what is needed to reach `pass` or its
  offline copy (laptop disk encryption and login, the GPG passphrase,
  the passphrase of the offline media) and the password manager's master
  password.

`pass`, the GPG secret key and `~/.ssh` are copied to two
LUKS-encrypted offline media, stored apart. Paper copies are kept in two
places as well. A medium and the paper that opens it are not stored
together off-site.

The catalog of these credentials (name, kind, storage class, rotation
trigger, procedure; never values or locations) lives in
`secrets-manager`'s `runbooks/outside-secrets-manager.md`. Credentials
that stay silently usable when stolen get a scheduled entry in
`rotation-schedule.json` (id prefix `pass:`); memorised passphrases are
rotated only on suspicion or device change. A yearly
`drill:offline-recovery` restores from one medium on another machine.

## Consequences

- The invariant gains exactly this exception; a new credential outside
  Secrets Manager needs a catalog entry, otherwise it doesn't belong
  there.
- The offline media go stale: after a change to `pass` or `~/.ssh`, both
  need a refresh.
- Losing the laptop is recoverable from one medium plus paper. Losing
  all paper copies and memory of the GPG passphrase is not.
- Values from `pass` are read one entry at a time and piped, never
  printed, the same rule as for Secrets Manager.
