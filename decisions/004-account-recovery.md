# ADR-004 — Account recovery when every passkey is lost

**Status:** Proposed

## Context

brig·id is passkey-only. An account whose passkeys are all lost is locked out for good
([core#52](https://github.com/brig-id/core/issues/52)). Any recovery path is a second way
in, so it must not become a weaker equivalent of a password.

## Decision (proposed)

- **Recovery code, not a user-chosen passphrase.** At setup the server generates 128 bits of
  randomness, shown once as a 12-word list (or grouped base32), and stores only a verifier:
  `HKDF-SHA3-256(MASTER_KEY-derived key, code)` per account, encrypted like other account
  fields. A user-chosen passphrase would be low-entropy and need a slow hash; a generated
  code does not.
- **What it unlocks: enrolling one new passkey, nothing else.** No session, no token, no
  access to identities before that passkey exists. The flow is
  `recovery/begin { username, code }` → WebAuthn registration challenge →
  `recovery/finish` stores the new passkey, then the code is **consumed** and a new one must
  be generated from the account.
- **Guards:** the same `/auth/*` rate limit plus a per-account attempt counter with lockout;
  every use is recorded so the account page can show "recovered on …"; constant-time
  comparison of verifiers.
- **Interaction with the last-passkey guard (core#41):** unchanged. Recovery adds a passkey,
  it never removes one.
- **Optional.** An account without a recovery code keeps today's behaviour.

## Consequences

- Rotating `MASTER_KEY` must re-encrypt the verifiers (same as other account fields).
- Identity DIDs and VSIDs are untouched by recovery: the account regains access to the same
  identities.
- To decide before implementing: whether a server operator may reset an account manually
  (support channel), which this ADR does not allow.

## References

- core#52, core#41, spec#8
