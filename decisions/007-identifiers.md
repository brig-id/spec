# ADR-007 — Identifiers: UUIDs for accounts and identities, a DID for the server only

**Status:** Accepted (2026-11-11, security audit finding S3)
The server's own DID document is removed by [ADR-025](025-server-surface.md) §1; the rest
of this record holds.

## Context

Accounts carry a DID `did:web:<host>:u:<username>` and identities a DID
`did:web:<host>:i:<uuid>` ([ADR-002](002-accounts-and-identities.md)). Neither is resolvable:
no route publishes their documents, and identity DIDs must never be published, since one is
the same at every relying party. A DID nobody can resolve is a UUID with a prefix. The account
DID also contains the username, which is now hidden and must stay renamable
([ADR-003](003-usernames-and-quick-signup.md)).

"VSID" reads *Verifiable Subject Identifier* in the spec and *Virtual Stable Identifier* in
the code; nothing about it is verifiable by a relying party beyond the token's signature.
`PrivateAlias::to_did_peer` builds a syntactically valid `did:peer:2` whose "Ed25519 key" is a
hash of the alias: a meaningless DID, in code no path uses.

Federation does not need per-user DIDs. A passkey works only on the domain that created it, so
a user of another server is sent to that server, which authenticates them. What a relying
party or another server must verify is a token signed by that server: it needs the **server's**
key, published in its JWKS and its `did:web` document.

No relying party exists yet ([ADR-005](005-account-sessions-and-rp-tokens.md) is not
implemented), so the VSID formula can still change without breaking any account anywhere.

## Decision

- **Accounts and identities are identified by their UUID.** No account DID, no identity DID.
- **The server keeps its DID** (`did:web:<host>`, `/.well-known/did.json`), which publishes the
  key that signs its tokens, alongside the JWKS.
- **VSID = Virtual Stable Identifier**, one name everywhere:

  ```text
  VSID = HKDF-SHA3-256(
      ikm  = vsid_secret,                          // ADR-006
      salt = none,
      info = "brigid-vsid-v2" ||
             identity_uuid (16 bytes) ||
             u32_be(len(client_id)) || client_id
  )[..32]
  → unpadded base64url (43 chars)
  ```

  The identity UUID has a fixed length and `client_id` is length-prefixed, so the
  concatenation is unambiguous; `v2` keeps it apart from the DID-based `v1`. Nothing in it
  depends on the username, the host name or the master key: renaming the account, moving the
  server to another domain or rotating the master key changes no VSID.
- **Removed:** the account's `did_web` column, the identity's `did` column,
  `PrivateAlias::to_did_peer` and the `did:peer` code, `RootId::to_did_web` for accounts.

## Consequences

- `protocol.md` §2 invariants become: same `(identity, client_id)` → same VSID; different
  `client_id` → different VSID; derived from an identity UUID only, never from an account,
  a username or a DID.
- Account session tokens no longer carry a VSID-like `sub`; with ADR-005 the account session
  is a server-side session, and only relying-party tokens carry a VSID.
- The VSID computed today from identity DIDs changes. No relying party holds one, so this is
  the moment to do it; after ADR-005 ships, it would be a migration.
- To build in core: the new formula, the column removals (migration), the dead DID code.

## References

- Security audit 2026-11-11, finding S3; ADR-002, ADR-003, ADR-005, ADR-006
