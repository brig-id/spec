# ADR-023 — What describes a person is encrypted end to end, with a key only their passkey gives

**Status:** Accepted as an objective of v1 (2026-11-13, review round 4 with the owner). The
design is completed at step 2 of v1, on a prototype and on measurements; it is built at
step 3, before the audit ([ADR-014](014-three-products.md) §4).

## Context

`brigid-store` encrypts sensitive fields with keys derived from the server's master key
([`security-model.md`](../security-model.md) §4.5). That defeats a stolen database. It does
not hide anything from whoever runs the server, who holds the key, nor from anyone who
obtains the key and a copy of the data.

A product that says "you do not have to trust us" should not ask its users to trust their
server's operator with their names. The WebAuthn **PRF** extension lets an authenticator
compute a secret from a passkey and a salt, at sign-in; the server never sees it.

## Decision

### 1. Scope: everything that describes

Encrypted in the browser, readable by the person only:

- the **labels** of identities: name, icon, description;
- the **attributes** of identities: given name, family name, username, picture, e-mail,
  language, time zone ([ADR-022](022-identity-attributes-and-consent.md)).

The server keeps in a form it can read only what it must know to work: identifiers, which
identity belongs to which account, which identity an account presents to which application,
passkeys, sessions.

### 2. Only passkeys that can do it are accepted

**A passkey that does not support PRF is refused, everywhere, from sign-up.** One level of
protection, with no weaker mode to explain, test or audit. This amends
[ADR-009](009-passkeys-only-user-verification.md): a passkey is no longer accepted
"whatever holds it".

What this excludes, by a third-party matrix dated September 2026: Windows Hello before the
Windows 11 update of February 2026 (Windows 10 altogether), and some password managers.
What it keeps: iCloud Keychain (iOS 18, macOS 15 and later), Google Password Manager,
recent Windows 11, security keys with `hmac-secret`, 1Password, Proton Pass.

The prototype of step 2 measures this on real devices before the rule is enforced. If the
exclusion proves much wider than this table, the rule comes back to the owner first.

### 3. Keys

- One random **data key** per account encrypts the fields of §1.
- It is stored once per passkey, wrapped by a key derived from that passkey's PRF output,
  and once more wrapped by the recovery code (§4). Adding a passkey requires a session that
  holds the data key.
- Synced passkeys carry their PRF secret with them, so they work on every device they reach.

### 4. Recovery

**The recovery code also decrypts.** It is split in the browser into a part that proves the
right to recover, sent to the server, and a part that unwraps the data key, never sent.
This amends [ADR-004](004-account-recovery.md), where the whole code reaches the server.

Someone with neither a passkey nor a recovery code has lost the account: no administrator
can give access back ([ADR-018](018-app-structure.md) §4), and nothing can decrypt what
described its identities. The interface says so at sign-up, when it offers the code.

### 5. Sharing with an application

An attribute a person agrees to share must reach the application. The browser decrypts it
on the linking page and hands it to the server for that sign-in; the server puts it in the
token and does not keep it. A sign-in without any page (`prompt=none`) still runs in the
browser, which does the same.

### 6. What it protects against, and what it does not

| Protects against | Does not protect against |
| --- | --- |
| A copy of the database, with or without the master key | An operator who changes the page the server serves, to capture the key |
| A curious administrator reading stored data | The server knowing which identities exist, whose they are, and where and when they sign in: it issues the tokens |
| A request to hand over stored data | What an application was given: it holds it |

The interface is served by the operator: end-to-end encryption in a web page raises the
cost of spying from "read a table" to "ship a modified page", which leaves traces and can
be detected. It is not a defence against an operator determined to cheat, and brig·id does
not say that it is.

## To settle with the prototype (step 2)

Design points, not questions for the owner unless a measurement contradicts §2:

- whether PRF is evaluated at creation or needs a first sign-in, per authenticator;
- the derivation (salts, `info` labels) and the format of a wrapped key;
- how the few existing accounts move to the new scheme;
- what the server can still validate (lengths, not contents) and what moves to the browser;
- pictures: size limits and storage of encrypted blobs;
- how a person is told, before creating a passkey, that their device cannot be used.

## Consequences

- `core`: wrapped keys per passkey and per recovery code; encrypted fields stored as opaque
  values; sign-up and passkey enrolment refuse a credential without PRF; the recovery flow
  of ADR-004 changes.
- `app`: key handling in the browser, on the most sensitive pages; the linking page decrypts
  what is shared.
- Attributes are stored from step 2 in a form that this scheme can take over without a
  second migration.
- The audit's scope includes all of it.
- [`security-model.md`](../security-model.md) gains the table of §6 and loses "zero-trust
  storage" as a description of what the operator cannot read.

## References

- W3C Web Authentication Level 3, PRF extension; CTAP 2.1 `hmac-secret`
- PRF support matrix (Corbado, modified 2026-09-29):
  <https://www.corbado.com/blog/passkeys-prf-webauthn>
- Yubico, developer's guide to PRF:
  <https://developers.yubico.com/WebAuthn/Concepts/PRF_Extension/Developers_Guide_to_PRF.html>
- Summary of Tim Cappalli's warning on PRF without independent recovery:
  <https://lilting.ch/en/articles/passkeys-prf-extension-encryption-risk>
- [ADR-004](004-account-recovery.md), [ADR-009](009-passkeys-only-user-verification.md),
  [ADR-021](021-sharing-between-servers.md), [ADR-022](022-identity-attributes-and-consent.md)
