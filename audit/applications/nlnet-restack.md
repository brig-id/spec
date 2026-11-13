# NLnet proposal — draft

> **Working draft, not a submission.** Call: Restack fund, deadline **2027-01-03, 12:00 CET**
> (<https://nlnet.nl/propose/>). See [ADR-010](../../decisions/010-third-party-audit-funding.md).
>
> **Before sending:**
>
> 1. **Rewrite every answer in your own words.** NLnet requires proposals written by a
>    human. This draft was produced with a generative AI model from the project's
>    repositories and decision records. Answer "yes" to the question on generative AI and
>    describe that use (model, what it did, the prompt). Answer "yes" to the question on
>    AI-generated code in the existing codebase.
> 2. Ask NLnet (or check the call's guide) that the **security audit** support service
>    applies to Restack.
> 3. Check each character limit on the live form; the limits below are those read on
>    2026-11-13.
> 4. Fill what is between `[brackets]`.
> 5. Licence: AGPL-3.0-or-later for the servers, the core and the interface, LGPL-3.0-or-later
>    for the cryptography library ([ADR-020](../../decisions/020-licensing-and-pricing.md)).
> 6. A token is needed to submit: request it by email as the form explains, a few days
>    ahead.

---

## Fund

Restack Fund.

## Proposal title

brig·id: a self-hosted sign-in server with passkeys only and unlinkable identities

## Project website and repositories

- <https://brig.id>
- <https://github.com/brig-id> (`crypto`, `core`, `app`, `server-leaf`, `spec`)
- Specification and decision records: <https://github.com/brig-id/spec>

## Abstract — summary of the project and expected results (1,000 characters)

brig·id is a free, self-hosted sign-in server. People sign in with a passkey and nothing
else: no password, no username to type, no email address. The account is private and carries
no profile. What a website meets is an identity the person chose for it, and each website
receives a different identifier, so two sites cannot tell that two identities are the same
person. A small server (one executable) lets a person, a family or an association be their
own identity provider instead of using "Sign in with Google".

Today the server exists (Rust, passkey accounts, identities, encrypted storage) but cannot
yet sign people in to other sites. This project delivers version 1: the standard OpenID
Connect flow for websites, tested end to end with a demonstration site; an accessible
interface with no third-party dependency; a single executable with
built-in TLS; and an independent security audit, with its findings fixed and published.

## Requested amount

EUR `[48,000]` (first proposal: 5,000 to 50,000).

## Budget breakdown — tasks, effort, rate (4,000 characters)

Rate: EUR `[400]` per day, `[one developer]`. 120 days.

| # | Milestone | Result | Days |
| --- | --- | --- | --- |
| 1 | OpenID Connect for websites | Authorization Code flow with PKCE, registered clients (with a secret, or public), identity selection, a per-site identifier as the only thing a site receives; specification updated | 25 |
| 2 | Demonstration site and end-to-end tests | A small relying party, written with an ordinary OpenID Connect library, signs in through a real passkey ceremony on every change; it doubles as the example for integrators | 10 |
| 3 | Self-contained, accessible interface | Own design tokens and components (dark and light, WCAG 2.2 AA), free icons and fonts embedded, no CDN and no paid dependency; accessibility checks in CI | 25 |
| 4 | Administration | First administrator through a one-time setup link, accounts, administrators, registration of websites, sign-up policy | 15 |
| 5 | One executable | Interface embedded, certificates by ACME, static binaries for x86-64 and arm64, checksums, SBOM and build provenance | 15 |
| 6 | Releases and supply chain | Automated releases from tags, dependencies pinned to tags and updated by pull requests | 5 |
| 7 | Documentation | Operator guide, guide for connecting a website, specification for auditors up to date | 10 |
| 8 | Security audit | Preparation with the auditors, fixes for every finding, publication of the report | 10 |
| 9 | Accessibility audit | Fixes for the findings | 5 |

Expenses: none.

## Comparison with existing or historical efforts (4,000 characters)

Free identity providers exist and are good: Keycloak, authentik, ZITADEL, Authelia, Kanidm,
Rauthy, Ory, Logto. They are built for organisations: one user is one profile, shown the
same to every application, and passkeys are one option next to passwords. They answer "how
does an organisation manage its users". brig·id answers "how does a person sign in to
websites without being followed from one to the next".

Pocket ID is the closest in spirit: small, self-hosted, passkeys only, and it works today.
It has one profile per user; we found no per-site identifier in its documentation.

What is different in brig·id:

- **The account is not what websites see.** It has no profile at all. A person keeps several
  identities (name, picture, description) and chooses which one a site meets.
- **A different identifier per site and per identity**, derived with a secret the server
  keeps. Pairwise identifiers exist in OpenID Connect; here they are the only kind, and they
  separate a person's own identities from each other as well.
- **Passkeys are the only way in**, with the user verified at each sign-in. No password to
  fall back to, so nothing to phish or leak. Recovery is a one-time code that can only enrol
  a new passkey.
- **Sized for a person.** One executable with an embedded database and its own TLS, for
  people who run one small machine. Larger editions come later.

Sign in with Google, Apple or Facebook are what most people use. They are convenient and
they see every sign-in. IndieAuth, Solid, Nostr and Sign-In with Ethereum give control to
the person but tie identity to one public identifier (a domain, a WebID, a key), which is
the opposite of unlinkability.

`[Say whether you considered contributing to one of these projects, as the form asks.]` The
separation between a private account and several unlinkable identities is the data model
itself; it cannot be added as an option to a server built on "one user, one profile".

`[The origin, in your words: the research on digital identity the idea comes from, and why
passkeys make it practical now.]`

## Technical challenges (4,000 characters)

- **A standard flow with a non-standard subject.** Websites must be able to use an ordinary
  OpenID Connect library, while receiving nothing but an identifier that is theirs alone.
  The project keeps choices stricter than the baseline (PKCE for every client, EdDSA
  signatures only); each has an interoperability cost to measure and document.
- **Unlinkability that holds.** The identifier a site receives must not let it link two
  identities, including through side channels: claims released, timing, the identity
  selection step, error messages, logs. The specification states the invariants; the audit
  must try to break them.
- **Passkeys only, for everyone.** No password fallback means the recovery path, the
  onboarding and the "wrong device" cases must be right for people who do not know what a
  passkey is. Accessibility (WCAG 2.2 AA) is part of that.
- **A first administrator without a first password.** A fresh server has no account and no
  email. The server prints a one-time setup link; until it is used, sign-up is closed.
- **One file that is safe by default.** Certificates obtained by itself, secrets encrypted
  at rest, a strict Content Security Policy with no third-party origin, a static binary that
  does not depend on the host's libraries.
- **Storage an operator cannot leak by accident.** Sensitive fields are encrypted before
  they reach the database; the audit must confirm that a copy of the database reveals
  nothing useful.

## Ecosystem — actors, dependencies, users (2,000 characters)

- **Standards**: WebAuthn (W3C), OpenID Connect (OpenID Foundation), ACME (IETF). The
  project implements them and tests against the OpenID Foundation's suite. Post-quantum
  hybrids follow IETF drafts and are checked against their test vectors.
- **Dependencies**: Rust crates from the RustCrypto, rustls, `webauthn-rs`, axum and sqlx
  projects; Qwik for the interface. Issues and fixes found on the way go upstream.
- **Users**: people and families who self-host; associations and non-profits that want
  sign-in for their members without a commercial identity provider; maintainers of
  self-hosted applications, who get a small provider to recommend and test against.
- **Engagement**: public specification and decision records written for reviewers; a public
  server run by the project (`helm.brig.id`) for trying it; packaging for Nix and
  distributions once the single executable exists; presentation to the self-hosting and
  identity communities (`[venues you intend: FOSDEM, forums…]`).

## Background — relevant experience (2,000 characters)

`[Your own words. Elements you may want: your work in human-computer interaction since 2008;
the 2009 paper on digital identity; your professional experience; that brig·id was restarted
in 2026 and what exists today: the cryptography library with known-answer tests and fuzzing,
the core with WebAuthn, sessions and the encrypted store, the leaf server with releases and
a deployed instance, the specification, an internal security review whose findings were
fixed.]`

## Other funding sources (1,000 characters)

None. The project is self-funded and has no revenue. `[If you send the OSTIF request, say
so: no funding obtained, a request for audit sponsorship in progress.]`

## Generative AI

- **Used in writing this proposal:** yes. `[Model, how it was used, and the prompt and
  output, as the form asks. Keep the logs.]`
- **Existing codebase contains AI-generated code:** yes. `[High-level description, 1,000
  characters: for example, that code and documentation are written with an AI coding
  assistant under your direction and review, that every change goes through a pull request
  you merge, with tests, static analysis, dependency audit and fuzzing in CI.]`

## Contact

`[Name, email, phone, country. Applicant type: individual, or your organisation.]`
