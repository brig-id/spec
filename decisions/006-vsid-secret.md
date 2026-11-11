# ADR-006 — The VSID secret is stored, not derived from the master key

**Status:** Accepted

## Context

A VSID is `HKDF-SHA3-256(salt, "brigid-vsid-v1" ‖ len ‖ identity_did ‖ len ‖ client_id)`
([`protocol.md`](../protocol.md) §2), and `salt` is derived from `MASTER_KEY`. Rotating
the master key therefore changes the `sub` of every identity at every relying party: each
person loses each account they hold there.

The master key is rotated when it may have leaked. That is exactly when the operator
cannot afford the rotation to destroy every relying-party account, so the design pushes
towards not rotating.

OIDC signing keys had the same coupling and lost it: they are stored in the database,
encrypted with a key derived from the master key, and a rotation re-encrypts them.

## Decision

- The VSID secret is **32 random bytes generated once**, at the first start of a new
  database, and stored encrypted with a key derived from the master key (HKDF context
  distinct from every other stored field).
- `VSID = HKDF-SHA3-256(ikm = vsid_secret, info = …)`: the formula and its `info` are
  unchanged, only the origin of the keying material changes.
- `leaf rotate-key` re-encrypts the VSID secret like every other stored secret. **VSIDs
  survive a master key rotation.**
- **Upgrade of an existing database:** when no VSID secret is stored, the value the server
  derived until now (`HKDF(MASTER_KEY, "brigid-vsid-salt")`) is stored as the secret. No
  VSID changes at the upgrade.
- There is no command to change the VSID secret. Changing it is the event this ADR
  removes from routine operations; if it is ever needed, it is a deliberate, announced
  migration.

### What a leaked master key means for VSIDs

Whoever holds the master key **and** a copy of the database can decrypt the VSID secret
and the identity DIDs, and so compute the VSID of any identity at any relying party. They
can already read which identities belong to which account, so this adds no linkage they
do not have. Rotating the master key stops later copies of the database from being
readable; it does not, and did not before this ADR, undo what was read.

An operator who judges that the relying-party identifiers themselves must change after an
incident is in the "deliberate migration" case above.

## Consequences

- `protocol.md` §2: the salt paragraph and invariant 5 change; `operations.md` §1 and §3
  lose the warning that a rotation changes every VSID; the audit checklist item on that
  warning goes.
- `core`: `brigid-store` stores and re-encrypts the secret; `derive_vsid_salt` remains only
  as the upgrade path.
- `server-leaf`: reads the secret from the store at startup; `rotate-key` no longer prints
  the VSID warning.
- Still tied to the VSID: the server's domain, through the identity DID
  (`did:web:<host>:i:<id>`). Moving an instance to another domain changes every VSID.
  Whether the VSID input should be the identity's id alone is left to the decision on what
  DIDs are for.

## References

- [ADR-002](002-accounts-and-identities.md); `protocol.md` §2; `operations.md` §1, §3
