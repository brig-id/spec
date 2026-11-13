# ADR-009 — Authentication: passkeys only, user verification required, no attestation

**Status:** Accepted (2026-11-13)

## Context

brig·id has only ever offered passkeys. That was a fact of the code, not a written decision,
and three questions kept coming back: may another method be added, must the user be verified
by a biometric, and should the server check which authenticator it is talking to.

## Decision

### 1. A passkey is the only way to authenticate an account

No password, one-time code, magic link, SMS, social or federated sign-in for the account, and
no "trust this device" that skips the passkey. The recovery code
([ADR-004](004-account-recovery.md)) is not an authentication method: it enrols one new
passkey and opens nothing by itself.

A passkey is a discoverable WebAuthn credential, whatever holds it: Windows Hello, Touch ID
and Face ID, an Android screen lock, a security key with a PIN or a fingerprint reader
(YubiKey Bio), a password manager.

### 2. User verification is required, everywhere

- `userVerification: "required"` in every ceremony: signup, sign-in (with or without passkey
  autofill), adding a passkey, enrolling after recovery, and any later re-authentication.
- The server refuses a response whose authenticator data does not carry both flags **UP**
  (user present) and **UV** (user verified).
- As built: `brigid-webauthn` uses the `webauthn-rs` passkey API
  (`start_passkey_registration`, `start_passkey_authentication`,
  `start_discoverable_authentication`), which sets `UserVerificationPolicy::Required` and
  checks the flag. To add: a test that an assertion with UV unset is refused, so a library
  upgrade cannot relax it silently.

Each sign-in is therefore two factors in one gesture: the device that holds the key, and
what unlocks it. brig·id needs no second step on top.

### 3. What "verified" means, and what it cannot mean

brig·id promises: *the authenticator verified a human, by a biometric or a PIN*. It never
promises a biometric, because WebAuthn cannot tell the two apart:

- The authenticator reports one bit, UV. It means "the user was verified by a method this
  authenticator supports". The method is not part of the signed data.
- The `uvm` extension (WebAuthn Level 2 §10.3) would name the method, but browsers and passkey providers do not
  implement it in practice, and its value is the authenticator's own statement: without
  trusted attestation (§4) it proves nothing.
- The FIDO Metadata Service lists what a model *can* do, not what happened in a ceremony.
- Every platform falls back to the PIN or passcode when the biometric fails (wet finger,
  mask, several failed attempts). A relying party cannot forbid the fallback.
- The biometric does not make the credential stronger for the server: both unlock the same
  private key, locally. The match never leaves the device.

Interface text says "your fingerprint, your face or your screen lock", never "biometrics".

### 4. Attestation is not requested

Registration keeps `attestation: "none"`.

- **Requiring attestation would exclude synced passkeys.** Passkeys synced by iCloud
  Keychain, Google Password Manager and most password managers come without an attestation
  statement: there is no single device to attest. They are what most people have. An
  attestation policy would accept hardware-bound authenticators only.
- **Privacy.** Attestation discloses the make and model of the authenticator. A server whose
  purpose is to learn as little as possible has no use for it.
- **The AAGUID stays a hint.** Without attestation it is unverified. brig·id uses it only to
  suggest a name for the passkey ([ADR-003](003-usernames-and-quick-signup.md)), never for a
  decision.

What is given up: the server cannot enforce "certified hardware only". If an organisation
needs that, it is a per-server policy to design later, off by default, in its own ADR.

## Consequences

- An authenticator that cannot verify its user cannot be used: U2F-only keys (which cannot
  hold a discoverable credential anyway) and FIDO2 keys with no PIN set (the browser asks to
  set one).
- No "MFA" setting, no second-factor enrolment, no SMS provider: there is nothing to add.
- If an `amr` claim is ever issued, it cannot say `fpt`, `face` or `pin`: the server does not
  know.
- The attestation code path of `webauthn-rs` (and the OpenSSL dependency it brings) is never
  exercised; see [ADR-013](013-leaf-single-executable.md).

## References

- W3C Web Authentication: authenticator data flags (§6.1), user verification requirement,
  attestation conveyance preference; Level 2 §10.3 for the `uvm` extension
- FIDO Alliance, synced passkeys and attestation:
  <https://fidoalliance.org/passkeys/>
- [ADR-003](003-usernames-and-quick-signup.md), [ADR-004](004-account-recovery.md);
  [`protocol.md`](../protocol.md) §3–§4
