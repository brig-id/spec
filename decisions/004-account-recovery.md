# ADR-004 — Account recovery when every passkey is lost

**Status:** Accepted (amended 2026-11-11 after the security audit, finding S5)

## Context

brig·id is passkey-only. An account whose passkeys are all lost is locked out for good
([core#52](https://github.com/brig-id/core/issues/52)). Any recovery path is a second way
in, so it must not become a weaker equivalent of a password.

## Decision

- **Recovery code, not a user-chosen passphrase.** The server generates it; a user-chosen
  passphrase would be low-entropy and need a slow hash, a generated code does not.
- **The code names its account.** Usernames are hidden ([ADR-003](003-usernames-and-quick-signup.md)),
  so the user can't be asked for one. The code is a random **recovery id** (64 bits) followed
  by the **secret** (128 bits) and a check byte, shown as 40 base32 characters in groups of
  four (`XXXX-XXXX-…`),
  to copy or download as a file. The recovery id is looked up through a keyed index, like
  usernames; it is not the account id nor the username, so renaming the account does not
  invalidate the code, and the code reveals nothing about the account.
- **Stored:** the keyed index of the recovery id (recomputed when the master key rotates,
  like `username_index`), and a verifier of the secret, `HKDF-SHA3-256(ikm = secret,
  info = "brigid-recovery-verifier-v1")`, both encrypted with the account's key. Never the
  code. The verifier needs no key: the secret has 128 random bits, so it can't be guessed
  from the verifier, and an unkeyed verifier survives a master key rotation without the
  secret. A check byte at the end of the code reports typos before any request.
- **What it unlocks: enrolling one new passkey, nothing else.** No session, no token, no
  access to identities before that passkey exists. The flow is `recovery/begin { code }` →
  WebAuthn registration challenge → `recovery/finish` stores the new passkey, then the code
  is **consumed** and the user is asked to generate a new one.
- **Guards:** the `/auth/*` rate limit, and **no per-account lockout**: a lockout would let
  anyone block a victim's recovery, and 128 bits of secret make guessing hopeless anyway.
  Constant-time comparison of verifiers. Every use is recorded, so the account page shows
  "recovered on …".
- **When:** the code is offered at the end of signup (onboarding) and can be regenerated
  from the account; generating one replaces the previous one.
- **Interaction with the last-passkey guard (core#41):** unchanged. Recovery adds a passkey,
  it never removes one.
- **Optional.** An account without a recovery code keeps today's behaviour.

## Consequences

- Rotating `MASTER_KEY` must re-encrypt the verifiers (same as other account fields).
- Identity DIDs and VSIDs are untouched by recovery: the account regains access to the same
  identities.
- A server operator cannot reset an account: an account without passkey and without recovery
  code is lost, and is deleted and created again ([ADR-018](018-app-structure.md) §4).

## References

- core#52, core#41, spec#8
