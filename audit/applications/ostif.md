# OSTIF audit request — draft

> **Working draft, not a submission.** OSTIF takes requests through its intake form
> (<https://ostif.org/get-an-audit/>), then an introductory call and a scoping
> questionnaire. See [`audit/README.md`](../README.md).
>
> **Before sending:** rewrite it in your own words; fill what is between `[brackets]`; be
> plain about the stage of the project (pre-1.0, one deployment): OSTIF prioritises by
> impact, and an honest early contact is worth more than an inflated one. Tell NLnet and
> OSTIF about each other.

---

## Project

**brig·id** — a free, self-hosted sign-in server (identity provider). People sign in with a
passkey only. The account is private; websites meet an identity the person chose, and each
website receives a different identifier, so identities cannot be linked.

- Website: <https://brig.id>
- Source: <https://github.com/brig-id> — `crypto`, `core`, `app`, `server-leaf`
- Specification for auditors: <https://github.com/brig-id/spec>
- Licence: AGPL-3.0-or-later (the cryptography library: LGPL-3.0-or-later)
- Languages: Rust (server, about `[N]` lines), TypeScript (interface, Qwik)
- Maintainer: `[name, contact]`

## Stage, plainly

- Pre-1.0. Released: `leaf` 0.3 (passkey accounts, identities, account sessions, encrypted
  storage). Not built yet: the OpenID Connect flow that signs people in to other websites.
- One public deployment, the project's own. No known third-party deployment.
- Version 1 is planned for `[date]`. **The audit is wanted on the v1 release candidate**, not
  on today's code. We are writing now to prepare it properly and to learn what would make
  the project a good candidate.

## Why audit it

An identity provider is the one component whose failure opens every other. This one makes
privacy claims (unlinkable identifiers, encrypted storage, passkeys with no fallback) that
people will rely on without being able to check them. The project is small enough to be
audited completely, and it is meant to be run by people who cannot audit it themselves.

## Proposed scope

| Component | What to review |
| --- | --- |
| `crypto` | Authenticated encryption of stored fields, key derivation, hybrid post-quantum primitives (X-Wing, composite ML-DSA) |
| `core` — WebAuthn | Registration and authentication ceremonies, user verification, challenge handling, recovery code |
| `core` — sessions | Server-side session in a cookie, same-origin enforcement, expiry |
| `core` — OpenID Connect | Authorization Code flow, PKCE, client registration, the token endpoint, signing keys and rotation |
| `core` — identifiers | Derivation of the per-site identifier; that two identities cannot be linked by a website or by two colluding websites |
| `core` — storage | That a copy of the database reveals no secret; master key rotation |
| `server-leaf` | Configuration, built-in TLS, headers and Content Security Policy, rate limits, the embedded interface |
| `app` | The sign-in pages: script loaded, handling of the passkey ceremony |

Out of scope: the operating system and the network of whoever runs it.

## What exists to help the auditors

- A threat model, the protocol description, a checklist of invariants to test and the
  decision records (`spec`).
- Tests, coverage, fuzz targets run in CI (`crypto`, `core`), `cargo audit`, `cargo deny`,
  SBOM on releases.
- Known-answer tests for the cryptographic constructions against the IETF drafts' vectors.
- For v1: end-to-end tests with a demonstration relying party and a real passkey ceremony.
- An internal review (November 2026) whose findings were fixed. `[Say whether you share its
  report.]`
- No previous third-party audit.
- Coordinated disclosure policy: `SECURITY.md` in each repository.

## Funding

The project has no revenue and cannot pay for an audit. A grant request to NLnet (Restack
fund, January 2027 call) includes a security audit through NLnet's own programme; `[state
the outcome when known]`. We ask OSTIF whether a sponsored audit, or a co-funded one that
complements NLnet's, is conceivable for a project at this stage, and what would have to be
true first.

## What we commit to

Fix every finding before or with the release, publish the report in full, and keep the
auditors' recommendations as tracked issues.
