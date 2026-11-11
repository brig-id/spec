# ADR-003 — Usernames and passkey-only quick signup

**Status:** Proposed

## Context

Before [ADR-002](002-accounts-and-identities.md), the account username was load-bearing: it
is part of the account's root DID (`did:web:<host>:u:<username>`), and every VSID was derived
from that DID. Renaming would have changed every `sub` already issued to every relying party,
which is why a passkey-only signup with an auto-generated, editable username
([spec#6](https://github.com/brig-id/spec/issues/6)) needed analysis first.

ADR-002 moves the VSID input to the **identity** DID (`did:web:<host>:i:<uuid>`), built from
an immutable id. The account DID no longer reaches any relying party.

Username-less sign-in now exists: `POST /auth/login/begin` without a username starts a
discoverable-credential challenge, and the user is taken from the passkey's user handle
([`protocol.md`](../protocol.md) §4).

## Decision

### Account username

- It is a **sign-in handle**, not an identifier anything external depends on: VSIDs, ID
  tokens and identity DIDs do not contain it.
- It stays unique per server (`username_index`) and keeps the existing syntax rules.
- Renaming it is allowed in principle. What a rename must update: the encrypted `username`,
  the `username_index`, and the account's `did_web` (whose document is not published to
  relying parties). Nothing else. The rename endpoint is not built yet.

### Quick signup (passkey only)

- Signing up without typing a username is allowed: the server generates one (readable,
  random, e.g. `swift-otter-4821`), the passkey is created as usual, and the user signs in
  later with username-less login. The username can be changed afterwards.
- Prerequisite met: username-less login (core#18) and passkey autofill (app#2).

### Identity usernames ("fixed or temporary", core#50)

- An identity has no username by default ([ADR-002](002-accounts-and-identities.md)): its
  name is a display name, and relying parties identify it by VSID.
- A username for an identity only exists if that identity signs in directly (later). It is
  then **fixed** once set, since another person could otherwise take the old handle and be
  mistaken for the first. A **temporary** handle is a generated one the server may recycle
  after the identity is deleted, never while it exists.

### Templates

Identity templates (core#51) only pre-fill the display fields (name, logo, description); an
identity does not remember its template. They carry no security meaning.

## Consequences

- spec#6 is answered: quick signup is safe with respect to VSIDs.
- `protocol.md` §2 invariant 4 ("never derived from a username directly") now holds
  structurally: the VSID input is the identity DID, which contains no name.
- To build: the quick-signup endpoint (username generation, collision retry), the account
  rename endpoint, and the UI for both.

## References

- spec#6, spec#8, core#50, core#51, core#18, app#2
- [ADR-002](002-accounts-and-identities.md)
