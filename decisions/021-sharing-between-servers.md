# ADR-021 — Identities across servers: no linked accounts; nothing shared between servers

**Status:** Accepted (2026-11-13, review round 4 with the owner).

## Context

The original idea: a server B aggregates the identities of a linked account on a server A.

> A is brig.id: *personal*, *family*, *temporary*. B is the employer: *designer*, *lead*.
> From B one would see the five; from A, the three. The purpose is one more layer of
> privacy: an identity of B that opens an SSH session cannot be used outside the company.

Linking the two accounts breaks the invariant the product stands on, *identities cannot be
linked to each other*: the employer would learn its employee's personal identities.

The track to analyse: **share some identities, not the account.** With an objection,
"everything is encrypted, so the employer sees nothing", and a counter-argument: today's
encryption protects against the theft of the database, not against the operator.

## Analysis

### 1. What the stated purpose needs

"An identity of B cannot be used outside the company" is **already true, and needs no
link**. An identity exists on one server. Only that server signs tokens for it, and only
the sites registered on that server accept them
([ADR-005](005-account-sessions-and-rp-tokens.md) §2). The employer controls which sites
are registered on B; nothing issued by B is worth anything elsewhere.

Two accounts on two servers, with no relation between them, give the wanted layer today.
What linking would add is not privacy. It is one screen instead of two.

### 2. The objection: "everything is encrypted"

It does not hold, for three reasons.

- **The operator holds the key.** `brigid-store` encrypts fields with keys derived from the
  server's master key ([`security-model.md`](../security-model.md) §4.5). That defeats a
  stolen database. The server itself decrypts names and descriptions to serve the
  interface; whoever runs it can do the same.
- **What matters is not in the encrypted fields.** If B *presents* an identity to a site
  (signs the token), B knows, in clear and by construction: that the identity exists, which
  account uses it, at which site, and when. No encryption of stored data removes what a
  party must know to do its job.
- **The link is the leak.** Whatever is shared into account *e* on B is tied to employee
  *e*. Two identities of A shared into the same B account are linked to each other and to
  the employee, even if B can read neither name.

### 3. Sharing identities instead of the account: three mechanisms

**(a) B copies the identity.** The identity becomes a B identity. B's operator can read it;
its identifiers at sites change (they derive from B's secret,
[ADR-007](007-identifiers.md)). This is "create an identity on B". Nothing to design.

**(b) B relays, A signs (identity brokering).** B is a relying party of A: A issues B a
token for the identity, B issues its own to the final site.

| | Learns |
| --- | --- |
| A | That the identity is used through B (so: this person has dealings with B), and when |
| B | A stable pseudonym of the identity; that it belongs to employee *e*; every site it is used at, and when; that two shared identities are the same person's |
| The site | A B identity. Nothing about A |

B learns everything except the name, and can impersonate the identity at every site that
trusts B. **Worse than doing nothing**: the personal identity's activity becomes visible to
the employer.

**(c) The browser presents, no server relays (portable credentials).** A issues a
credential the person's device holds and presents directly to sites (OpenID for Verifiable
Presentations, SD-JWT credentials, the browsers' Digital Credentials API). A does not see
where it is used; B is not involved at all. But every site must accept such credentials
instead of OpenID Connect; a credential shown at two sites is linkable unless one is issued
per site, which brings the issuer back in; and it is another protocol family to implement
and audit. A possible future, not a feature of this product today.

**In all three, B is either fully trusted with the identity or not involved.** There is no
useful middle where B aggregates an identity it cannot read and still presents it.

### 4. The direction was reversed

The server that should know the least about a person is the employer's. The scenario makes
it the place where everything is gathered. If anything aggregates, it must be on the side
the person controls:

- **In the browser.** The app remembers, in the browser's own storage, the servers this
  person uses, and offers them in a switcher ("brig.id", "my company"), each opening that
  server's own interface. No server learns of another. Stored per browser, not synced:
  that is the price of nobody knowing.
- A personal server holding links to others is a weaker variant (the personal server learns
  the employer; acceptable on one's own seedling, not on a shared server).

"From B I see five" becomes "in my browser I see my two servers". The employer's server
keeps seeing two.

### 5. End-to-end encryption with a key only the user holds

Looked at here as a way to make sharing safe, which it is not: it can hide an identity's
name, description and attributes from the operator, and cannot hide what §2 says matters.
It is worth having for itself, and the review made it an objective of v1:
[ADR-023](023-end-to-end-encryption.md).

### 6. Revocation

Nothing is shared, so nothing is revoked between servers. On each server: deleting an
identity ends it; leaving the company ends the account on B, and with it every B identity.
That is the property the scenario asked for.

### 7. SSH

A separate subject, for later, and it needs no link either: the company's server signs
short-lived SSH certificates for a B identity after a passkey sign-in, with the company's
own certificate authority (the pattern of OpenPubkey SSH and of `step-ca`). A certificate
from B's authority is accepted only by machines that trust B: unusable outside the company,
by construction. Its own ADR when the time comes.

## Decision

1. **Accounts and identities are never linked across servers.** A security invariant: a
   server holds no reference to an account or an identity of another server.
2. **No server-to-server sharing is built.** Separation by server is the feature, and the
   documentation and the site say so: an identity lives on one server and works only for
   the applications of that server.
3. **A single view, in the browser, after v1**: a list of "my servers" kept in the browser
   only. It builds on the "choose another server" step of sign-in
   ([ADR-003](003-usernames-and-quick-signup.md)).
4. **End-to-end encryption is an objective of v1**, for itself:
   [ADR-023](023-end-to-end-encryption.md).
5. **SSH certificates: after v1**, as a service of an identity
   ([ADR-022](022-identity-attributes-and-consent.md) §1b), with their own record.

**Not defined: "the brig·id network".** The idea of servers connected to a network (a
directory of servers, a federation, or both) exists and has no definition yet. It is a
subject of analysis after v1. Whatever it becomes, it must hold with point 1.

## Consequences

- No protocol between servers to specify, implement or audit in v1.
- [`security-model.md`](../security-model.md) gains the invariant of point 1.
- The origin idea, "from B one sees the five", is given up as stated.

## References

- W3C Web Authentication Level 3, PRF extension; CTAP 2.1 `hmac-secret`
- PRF support matrix (Corbado, modified 2026-09-29):
  <https://www.corbado.com/blog/passkeys-prf-webauthn>
- Summary of Tim Cappalli's warning on PRF and encryption:
  <https://lilting.ch/en/articles/passkeys-prf-extension-encryption-risk>
- Yubico, developer's guide to PRF:
  <https://developers.yubico.com/WebAuthn/Concepts/PRF_Extension/Developers_Guide_to_PRF.html>
- OpenID for Verifiable Presentations: <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html>
- OpenPubkey SSH: <https://github.com/openpubkey/opkssh>
- [ADR-002](002-accounts-and-identities.md), [ADR-004](004-account-recovery.md),
  [ADR-005](005-account-sessions-and-rp-tokens.md), [ADR-007](007-identifiers.md),
  [ADR-009](009-passkeys-only-user-verification.md)
