# ADR-024 — The life of an account: starting, recovering, leaving

**Status:** Accepted (2026-11-13, review round 5 with the owner). Amends
[ADR-002](002-accounts-and-identities.md) and [ADR-004](004-account-recovery.md).

## Context

Re-reading the earlier records against what the review decided showed four things that no
longer hold together, and three functions that do not exist.

- The server creates three starter identities at sign-up. With end-to-end encryption
  ([ADR-023](023-end-to-end-encryption.md)) it can no longer write their names.
- ADR-002 kept a door open: passkeys attached to one identity, opening a session limited to
  it. Nothing was built, and a data key per account makes it much harder.
- The recovery code is optional, while no administrator can give access back
  ([ADR-018](018-app-structure.md) §4) and losing every passkey now loses the data too.
- Nobody can delete their own account, take their data away, or stop new sign-ups on a
  server without touching its database.

## Decision

### 1. Starter identities are proposed, not imposed

At the end of sign-up the onboarding **proposes** starter identities (for example Personal,
Work, Public), ticked, that the person can rename, untick or complete before continuing.
The browser creates the ones kept. The server creates none.

### 2. A passkey always opens an account

The possibility of a passkey that opens one identity only is **closed**. The nullable
`identity_id` on credentials that ADR-002 reserved is not kept. A passkey is a way into an
account; identities are chosen inside it.

### 3. The recovery code is mandatory

Onboarding does not finish until the person has confirmed keeping their code (copied,
downloaded or printed). It can be generated again, which replaces it; it cannot be removed.
This replaces "Optional" in ADR-004. Existing accounts without a code are asked for one at
their next sign-in.

### 4. Leaving: delete the account, take the data

- **A person can delete their account**, after a passkey ceremony done for that purpose. It
  removes the account, its passkeys, sessions, identities, and what was recorded about the
  applications they signed in to. The page says what it cannot do: an application keeps
  what it received. The last administrator of a server cannot delete their account before
  naming another.
- **A person can export their data**: their identities with their labels and attributes,
  and the list of applications each one signed in to, in a documented, readable file. With
  end-to-end encryption the browser produces it.

### 5. Sign-up can be closed

- **From step 1 of v1**: a server setting closes the creation of accounts. The project's
  own server needs it before it has any administration.
- **At step 2**: the sign-up policy of the administration (open, by invitation, closed), and
  for an open server an **optional challenge** against automated sign-ups. It is
  self-hosted, like everything on the sign-in pages ([ADR-012](012-self-contained-app.md)):
  a proof-of-work challenge, no third-party captcha. Which one is chosen with the
  implementation.

### 6. Not in v1

Seeing and ending one's other sessions (today only the current one can be ended) comes
after v1.

## Consequences

- `core`: no starter identities at registration; the recovery code required to complete
  sign-up; account deletion; the sign-up setting; the credentials table loses the reserved
  column if it has it.
- `app`: the onboarding (identities proposed, code confirmed); the account pages for
  deletion and export.
- Steps: the sign-up setting at step 1; onboarding, mandatory code, deletion and export at
  step 2.

## References

- GDPR, articles 17 (erasure) and 20 (portability)
- [ADR-002](002-accounts-and-identities.md), [ADR-004](004-account-recovery.md),
  [ADR-018](018-app-structure.md), [ADR-023](023-end-to-end-encryption.md)
