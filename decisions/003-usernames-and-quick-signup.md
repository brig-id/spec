# ADR-003 — Usernames and passkey-only quick signup

**Status:** Accepted (amended 2026-11-11: the username is hidden)

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

- It is an **internal sign-in handle**, not an identifier anything external depends on:
  VSIDs, ID tokens and recovery codes do not contain it ([ADR-007](007-identifiers.md),
  [ADR-004](004-account-recovery.md)).
- **It is hidden from the user.** Every new account gets a generated one (readable, random,
  e.g. `swift-otter-4821`); sign-in uses the passkey alone (discoverable credential), so the
  user never types it. Existing accounts keep the username they chose.
- It stays unique per server (`username_index`) and keeps the existing syntax rules.
- **Renaming it must stay possible** (later, from the account settings) and breaks nothing:
  what a rename updates is the encrypted `username` and the `username_index`. Nothing else
  refers to it. The rename endpoint is not built yet.

### Signup

- Signup is **passkey only**: no username field. The server generates the username, the
  passkey is created, and the account exists only once the passkey is stored.
- The server opens the session at the end of registration: one passkey prompt, not two.
- Then a minimal onboarding page: the passkey's name, pre-filled with its provider when the
  authenticator says which one it is (AAGUID, e.g. "Google Password Manager", "1Password"),
  editable; and the recovery code ([ADR-004](004-account-recovery.md)), to copy or
  download.

### Sign-in

- The server is the one serving the page by default. Another server is chosen by its
  domain and the browser is sent to that server's sign-in page: a passkey only works on
  the domain that created it. A directory of public servers may come later.
- "Use my passkey" (discoverable credential) is the main action. Passkey autofill needs an
  input field, so it is offered only where one exists.

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
