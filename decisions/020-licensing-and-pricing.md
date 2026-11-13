# ADR-020 — Licences and pricing: AGPL with a commercial licence, paid modules kept apart

**Status:** Proposed (2026-11-13). A recommendation, deliberately not a decision: the choice
is the owner's.

## Context

Everything public is under **LGPL-3.0-or-later**. `server-forest` is private and has no
licence. The site says `leaf` and `grove` are free and `forest` is commercial.

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

## Recommendation

**B, with a small E.**

- **`core`, `app` and the three servers under AGPL-3.0-or-later.** One licence for the three
  servers, public, `forest` included. `core` must follow, or the servers' licence protects
  nothing.
- **`crypto` stays LGPL-3.0-or-later**: a thin library over public primitives gains from
  being reused and reviewed, and holds no business value.
- **A commercial licence** for the same code, for organisations that cannot use the AGPL,
  available for any edition and priced by edition. This is "a paid version for every
  edition" (intention 1), honestly named.
- **Paid, per edition**: support and guaranteed updates; the commercial licence; hosting.
- **Edition features (intention 3)**: nothing is gated inside AGPL code. A feature is either
  in the open product for everyone (customising the sign-in page, in the three), or a
  **separately licensed module** (replacing the whole interface, white-label). Rule: **no
  security feature is ever paid**. Everything that touches authentication, identities,
  tokens and cryptography is open and in the audit's scope.
- **A contributor agreement from the first outside contribution**, as a licence grant, not
  an assignment, with a public commitment that what is published under the AGPL stays under
  it. The FSFE's Fiduciary Licence Agreement is the model to start from.

Why not D, although it is what was asked: brig·id's argument is "you do not have to trust
us", and the right to run, change and leave is what makes the sentence true. And the audit
that the project needs most is funded by organisations that fund free software only.

If the requirement remains "a company must sign before using it", D is the honest way:
PolyForm Small Business, the words "source-available" on the site, and ADR-010 rewritten
without NLnet and OSTIF.

## Consequences if B is accepted

- **Relicensing is possible today by the owner alone**: the history has one human author
  (and automated dependency updates). It gets harder with each outside contributor: decide
  before the first one, and before the NLnet proposal, which asks for the licence. Note:
  part of the code was written with AI assistance, whose copyright status is unsettled in
  several jurisdictions; this does not prevent choosing a licence, it may weaken enforcing
  one.
- `core`, `app`, `server-*`: `LICENSE`, headers, `Cargo.toml` and `package.json`; released
  versions keep the licence they were published under.
- The site: the editions are no longer "free" against "paid". Each edition is free
  software; support, the commercial licence and hosting are paid. "Free for people" stays
  true (it becomes true for everyone).
- The backlog items "study relicensing from LGPL to AGPL" and "decide server-forest's
  licence" are closed by the decision.

## References

- GNU AGPL-3.0, section 7 (additional terms): <https://www.gnu.org/licenses/agpl-3.0.html>
- *Neo4j v. PureThink*: <https://sfconservancy.org/news/2025/jan/13/neo4j-amicus/>,
  <https://www.theregister.com/2025/02/27/adverse_appeals_court_ruling_could/>
- Functional Source License: <https://fsl.software>; Business Source License:
  <https://mariadb.com/bsl11/>; Elastic License 2.0: <https://www.elastic.co/licensing/elastic-license>
- PolyForm licences: <https://polyformproject.org/licenses>
- Zitadel's move to AGPL: <https://zitadel.com/blog/apache-to-agpl>
- FSFE Fiduciary Licence Agreement: <https://fsfe.org/activities/fla/>
- [ADR-010](010-third-party-audit-funding.md), [ADR-014](014-three-products.md),
  [ADR-019](019-editions.md)
