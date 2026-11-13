# ADR-002 — Accounts and identities

**Status:** Accepted

> **Read with what followed.** [ADR-005](005-account-sessions-and-rp-tokens.md): the account
> has a session, never a token; the token endpoint of this record is gone.
> [ADR-007](007-identifiers.md): identities have no DID; the VSID derives from the
> identity's UUID. [ADR-022](022-identity-attributes-and-consent.md): an identity's name
> and logo are labels for the person and are never sent to a relying party, which receives
> an identifier and the attributes the person accepted to share.
> [ADR-024](024-account-lifecycle.md): starter identities are proposed, not created by the
> server, and a passkey always opens an account, never one identity.
> [ADR-015](015-icons.md): logo references are `icon:ph:<name>`.

## Context

Until now a brig·id user is a single record (`users`): its root DID
(`did:web:<host>:u:<username>`) feeds the VSID, and `/auth/login/finish` issues an OIDC
token for it. The roadmap (core#49, app#11, app#27) needs several **identities** per person,
each with a name and a logo, and the web UI must let someone present a different face to
each service.

[`protocol.md`](../protocol.md) §2 forbids deriving a VSID from a "virtual identity". Taken
literally, two identities of one account would reach the same relying party (RP) with the
same `sub`, which defeats the point of having identities.

## Decision

### Two kinds of subject

- An **account** is private. It is never presented to an RP. Its passkeys give access to
  everything: the account's identities and its settings. At least one account passkey is
  mandatory.
- An **identity** is what an RP sees. It belongs to exactly one account. It is the
  Keycloak-style "user" of the RP's realm.

The server always knows which of the two it is dealing with. The login ceremony (WebAuthn,
DID) is the same for both.

### Identity record (table `identities`)

| Field | Notes |
| --- | --- |
| `id` | UUID v4, immutable, opaque |
| `account_id` | the owning account (`users.id`) |
| `did` | `did:web:<host>:i:<id>`: built from `id`, never from a name, so renaming changes nothing |
| `name` | display name, required, editable, encrypted like `username` |
| `logo` | required, a typed reference: `icon:<style>:<name>` from the predefined library, which lives in the app (the server checks the shape of the reference, not its presence in the library); `upload:<id>` is reserved for later |
| `description` | optional free text, encrypted |
| `created_at`, `updated_at` | |

`name` is a display name. It is **not** a login name and not unique. A login name
(`username@host`) for an identity is a separate, optional, later addition, only for an
identity that signs in directly.

### VSID

The VSID is computed from the **identity DID**, never from the account DID, an alias, a
virtual identity, a display name or a username:

```
VSID = HKDF-SHA3-256(salt, "brigid-vsid-v1" || len(identity_did) || identity_did || len(client_id) || client_id)
```

Two identities of one account get different VSIDs at the same RP, and the same identity
gets a stable VSID at each RP. The formula itself does not change; its input does.

### Sign-in

- **Now:** only account passkeys exist. Signing in with the account opens a session in
  brig·id. A token for an RP is issued for an identity the account chose, never for the
  account itself.
- **Later, forward-compatible:** a credential may optionally belong to an identity
  (nullable `identity_id` on the credentials table). Signing in with it opens a session
  scoped to that identity only, without access to the account. Not implemented yet; the
  schema must not prevent it.
- The UI offers an obvious control to switch between the account and the identities.

## Consequences

- `protocol.md` §2 invariant 3 and `security-model.md` §6 invariant 2 are rewritten: the VSID
  input is an identity DID.
- Existing users are accounts. They get the starter identities (Public, Personal, Work)
  at their next sign-in, like a new account at registration. For a token issued to an
  identity, **the VSID differs from the account's former one**, because the input changes:
  acceptable before 1.0 and while no RP depends on them, to be re-checked before any
  production RP is registered.
- The token claim `alias_type` carries no meaning once only identity tokens exist; to be
  removed with the core change.
- The flow by which an RP token is requested for a chosen identity (identity selection
  after account sign-in) is not specified here; it belongs with the app work (app#11).
- Rotating `MASTER_KEY` still changes every VSID, as before.

## Implementation

- `core` (brig-id/core#72): table `identities`, store, validation, identity DID,
  `/account/identities` (CRUD) and `/account/identity-templates`; starter identities
  (Public, Personal, Work) created at registration and, for older accounts, at the
  next sign-in.
- `app` (brig-id/app#32): Identities page, icon library.
- `core`: `POST /account/identities/{id}/token` issues an ID token whose `sub` is the VSID
  of that identity.
- Still open: the flow by which a relying party asks for that token (client registration,
  redirect, identity selection); the account session, which is still an ID token derived
  from the account DID (`/auth/login/finish`); and identity passkeys (nullable
  `identity_id` on credentials).

## References

- core#49: identity data model
- app#5, app#11, app#27
