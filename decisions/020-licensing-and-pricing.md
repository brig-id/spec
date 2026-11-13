# ADR-020 — Licences and pricing: AGPL alone, a registered name, paid services

**Status:** Accepted (2026-11-13, review round 3 with the owner).

## Context

Everything public is under **LGPL-3.0-or-later**. `server-forest` is private and has no
licence. The site says the two smaller editions are free and `forest` is commercial.

The intention now:

1. a paid version for **every** edition, not only `forest`;
2. **one licence for the three servers**, public, described as "AGPL, with commercial use
   forbidden without a contract";
3. **features by edition** (the three customise the sign-in page; only `forest` replaces the
   whole interface).

## Two facts first

**"AGPL, commercial use forbidden" does not exist.** The AGPL grants everyone the right to
run the program for any purpose. A text that adds "except commercially" is no longer the
AGPL: its section 7 lets a recipient remove such a "further restriction", and calling the
result open source is misleading (in *Neo4j v. PureThink*, a US court held exactly that
about AGPL plus the Commons Clause; whether the clause could be removed is before the Ninth
Circuit, with no decision found at this date). A licence is the AGPL, or it forbids
commercial use. Not both.

**A feature reserved to an edition inside AGPL code is not protected.** Anyone may remove
the check and redistribute. The same holds for a hard limit on accounts.

And one about today: under the **LGPL**, a library may be linked into a closed program.
Almost all of brig·id's logic is in `core`. Whatever the servers' licence, a third party can
build a closed server on `core` as long as `core` is LGPL.

## Options

| | A. LGPL, sell services | B. AGPL + commercial licence | C. Source-available, non-compete (FSL, BSL, Elastic v2) | D. Source-available, non-commercial (PolyForm Noncommercial or Small Business) | E. Open core (paid modules apart) |
| --- | --- | --- | --- | --- | --- |
| Must a company pay to use it? | No | No (it pays to avoid the AGPL, or for services) | No: internal use is allowed, only competing with us is not | **Yes** | For the modules |
| Open source (OSI)? | Yes | Yes | No | No | The core, yes |
| Eligible for NLnet and OSTIF ([ADR-010](010-third-party-audit-funding.md))? | Yes | Yes | No | No | The core, yes |
| Stops a competitor closing or hosting a modified version? | No | Largely (it must publish its changes to its users) | Yes | Yes | For the core, as its licence |
| Protects edition features? | No | No | By the licence terms | By the licence terms | **Yes**, by separation |
| Outside contributions | Simple | Need a contributor agreement | Need one | Need one | Need one for the modules |
| Trust, for a privacy product | High | High (the licence of Nextcloud, Mastodon, Bitwarden's server) | Lower: "you may read it" is not "you may leave with it" | Lower still | High if security stays in the core |
| Packaged by distributions | Yes | Yes | No | No | The core |

Seen in this field: Zitadel moved from Apache-2.0 to AGPL-3.0 in March 2025, with a
commercial licence and a contributor agreement; authentik is open core; Keycloak is
Apache-2.0 and lives on a vendor's services. Sentry created the FSL, which converts each
version to Apache-2.0 or MIT after two years.

## What the options mean for the intention

- **Intention 2, read literally, is option D.** It is the only family where a company must
  sign to use the software. It costs the open-source status, so the funded audit
  (ADR-010), distribution packaging, and part of what the site promises. "Commercial" also
  has to be defined (a non-profit with employees? a freelancer's own login? a school?), and
  every unclear case becomes a message to answer. PolyForm Small Business draws the line
  most cleanly (free below a size and revenue threshold).
- **Option C does not make companies pay.** It protects against a competitor, a risk that is
  theoretical today.
- **Option B makes no one pay by law**, but it creates the conversation: many companies
  forbid AGPL software by policy, and buy the same code under other terms. With B, what is
  sold is the commercial licence, support and updates, hosting, and the modules of E.
- A licence does not create revenue. It decides who has to talk to you.

## Decision

**AGPL alone. No commercial licence, ever.**

- **`core`, `app` and the three servers: AGPL-3.0-or-later.** One licence for the three
  servers, public, `forest` included. `core` follows, or the servers' licence protects
  nothing.
- **`crypto` stays LGPL-3.0-or-later**: a thin library over public primitives gains from
  being reused and reviewed, and holds no business value.
- **No dual licensing.** The project never sells an exception to the AGPL. Nobody, the owner
  included, can take the code closed: for a product that says "you do not have to trust
  us", this is the strongest form of the promise. What is given up: the organisations that
  forbid AGPL software by policy and would have bought other terms.
- **Contributions under a Developer Certificate of Origin**, nothing to sign: a contributor
  certifies the right to contribute, and contributes under the project's licence. With no
  other licence to grant, no contributor agreement is needed.
- **The name is registered.** "brig·id" (name and mark) is filed as a trademark before
  step 2 of v1 ([ADR-014](014-three-products.md)), and a short policy says what others may
  do with it: a fork or a host may say truthfully that it is based on brig·id or compatible
  with it; it may not call itself brig·id or suggest it is the project. A trademark does not
  stop anyone from running the software for free, nor a fork under another name; with the
  AGPL, which makes whoever offers a modified version as a service publish the changes, it
  is the realistic protection.

**What is paid, for any edition:**

- support and guaranteed updates;
- hosting by the project;
- **modules that are separate programs**, talking to a server through its interfaces: a
  replacement interface, white-label. A paid module cannot be a part linked into the server:
  the server contains contributors' code, which the project has no right to close.

**No security feature is ever paid.** Everything that touches authentication, identities,
tokens and cryptography is free software and in the audit's scope. Nothing is gated inside
the AGPL code, and no edition has a ceiling ([ADR-019](019-editions.md) §4).

**The change of licence happens now**, at step 1 of v1, together with the rename of the
smallest edition, which touches the same files.

## Why not the other options

- **D (non-commercial)** is what was first asked for: a company must sign to use the
  software. It costs the open-source status, so the funded audit
  ([ADR-010](010-third-party-audit-funding.md)), distribution packaging, and part of what
  the site promises; and "commercial" has to be defined case by case.
- **C (non-compete)** does not make companies pay and is not free software either.
- **B with a commercial licence** keeps one more thing to sell, at the price of a
  contributor agreement and of a standing possibility that the code be offered on other
  terms. The review chose the simpler promise.

## Consequences

- **Relicensing is done by the owner alone**: the history has one human author (and
  automated dependency updates). Released versions keep the licence they were published
  under. Part of the code was written with AI assistance, whose copyright status is
  unsettled in several jurisdictions; this does not prevent choosing a licence, it may
  weaken enforcing one.
- `core`, `app`, the servers: `LICENSE`, the manifests, and a `CONTRIBUTING` file stating the
  Developer Certificate of Origin, checked on pull requests.
- A trademark policy is published with the organisation's files; filing the mark is an
  owner action.
- The site: every edition is free software. Support, hosting and separate modules are paid.
  "Free for people" stays true, for everyone.
- The NLnet proposal names the AGPL.
- The earlier questions "study relicensing from LGPL to AGPL" and "decide server-forest's
  licence" are closed.

## References

- GNU AGPL-3.0, section 7 (additional terms): <https://www.gnu.org/licenses/agpl-3.0.html>
- *Neo4j v. PureThink*: <https://sfconservancy.org/news/2025/jan/13/neo4j-amicus/>,
  <https://www.theregister.com/2025/02/27/adverse_appeals_court_ruling_could/>
- Functional Source License: <https://fsl.software>; Business Source License:
  <https://mariadb.com/bsl11/>; Elastic License 2.0: <https://www.elastic.co/licensing/elastic-license>
- PolyForm licences: <https://polyformproject.org/licenses>
- Zitadel's move to AGPL: <https://zitadel.com/blog/apache-to-agpl>
- Developer Certificate of Origin: <https://developercertificate.org>
- EU trade mark regulation 2017/1001, article 14 (limits of the rights: honest referential use)
- [ADR-010](010-third-party-audit-funding.md), [ADR-014](014-three-products.md),
  [ADR-019](019-editions.md)
