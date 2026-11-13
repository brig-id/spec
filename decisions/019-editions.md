# ADR-019 — Editions: what leaf, grove and forest are made of

**Status:** Proposed (2026-11-13). Follows [ADR-014](014-three-products.md).

## Context

The working table for the three products (the maximum is what load tests must reach; ten
identities per account on average):

| | Accounts (target, tested maximum) | Deliverable | Machine | For |
| --- | --- | --- | --- | --- |
| leaf | 1–10 (50) | one executable | a micro-server, one service among others | tests, a personal server |
| grove | 10–100 (1,000) | a container image or a set of containers | a dedicated machine | non-profits, associations, free public servers |
| forest | 100–1,000 (1,000,000) | a set of pods and how to assemble them | a cluster | companies, very large non-profits, resellers |

Five questions were asked. They are answered in order.

## 1. Can leaf be "simpler than SQLite"?

No. SQLite is already what the question describes: a library compiled into the binary, and
one file.

| | SQLite (today) | redb | Encrypted flat files |
| --- | --- | --- | --- |
| What the operator sees | One file (plus two journal files while running) | One file | A directory |
| In the binary | A C library, about 1 MB, needs a C compiler to build | Pure Rust | Nothing |
| Queries | SQL; indexes and constraints declared | Key-value: every index (username, recovery id, session hash, expiry) written and kept consistent by hand | Read everything, filter in memory |
| Transactions, crash safety | The reference: tested to full branch coverage, in production everywhere for two decades | ACID, stable file format since 2023; far less exposure | None: two concurrent writes or a crash mid-write corrupt data |
| Tooling | `sqlite3`, online backup, continuous replication tools | None | `cp` |
| Cost of moving | — | Rewrite `brigid-store` (2,345 lines, the encrypted store, the most sensitive crate) and its 11 migrations; a data migration tool; and **two store implementations to keep identical for ever**, because forest needs SQL anyway | Same rewrite, to lose safety |

The only real gains of redb are a build without C and a slightly smaller binary. They do not
pay for a second implementation of the layer where a divergence is a security bug.

What can be simpler in `leaf` is what the operator touches, not the engine: no configuration
to write ([ADR-013](013-leaf-single-executable.md)), `leaf backup` producing one consistent
file while the server runs, and a clean stop leaving exactly one file.

**Recommendation: leaf keeps SQLite.**

## 2. What really distinguishes the three

Not volume. 1,000 accounts with 10 identities each are 10,000 rows, and an identity provider
is used a few times a day per person. With 25 operations per account and per day and a peak
at ten times the average:

| Accounts | Average | Peak |
| --- | --- | --- |
| 100 | 0.03 per second | 0.3 per second |
| 10,000 | 3 per second | 30 per second |
| 1,000,000 | 290 per second | 2,900 per second |

SQLite on a small machine handles the first two lines with a very wide margin. Only the last
needs more than one process. **The editions differ by who operates them and by what an
outage costs**, and the table follows from that:

| | leaf | grove | forest |
| --- | --- | --- | --- |
| Operated by | One person, for themself, a family, a test | A volunteer or a small team, for an organisation or the public | A platform team |
| Deliverable | One executable | One container image, a compose file | Stateless containers, a Helm chart, an integration guide |
| Storage | SQLite file | SQLite file on its own volume, **continuously replicated** to a second place | PostgreSQL, provided by the operator |
| Availability | Stops with the machine; restored from a backup | One node; restarts in seconds; documented recovery point and time | No single point of failure: two nodes or more, rolling updates without interruption |
| State in memory | Allowed | Allowed | None (§3) |
| TLS | Built in (ACME) | Built in, or a proxy | The cluster's ingress |
| Master key | A file or the environment | A container secret | The platform's secret manager |
| Sign-up | By invitation by default | Open or by invitation: a public server needs quotas per account and abuse controls | By policy |
| Administration ([ADR-018](018-app-structure.md)) | Minimal, plus the command line | Complete: invitations, policies, relying parties, audit log, appearance | Complete, plus integration: audit export, metrics, white-label |
| Updates | Replace the file | Follow the image tags (what `helm.brig.id` does) | Rolling, migrations as a job |
| Engine | **single node** | **single node** | **distributed** |

So: three products, **two engines**. `leaf` and `grove` share the single-node engine and
differ by packaging, operations and administration. `forest` is the other engine.

**grove on PostgreSQL and Redis** (what its README announces) is not recommended: for one
dedicated machine and a thousand accounts, two more services to run, back up and upgrade
buy nothing, and the person who runs a grove is often a volunteer. The argument for it, an
easier move from grove to forest, is covered by the export format (§6).

## 3. forest: stateless nodes and PostgreSQL

Yes. What must leave the process memory, and where it goes:

| State today | In forest |
| --- | --- |
| WebAuthn ceremonies in progress (`pending_registrations`, `pending_authentications`, in memory) | A PostgreSQL table with an expiry: inserted at `begin`, deleted and returned at `finish`. The database guarantees single use, on any node. At 2,900 operations per second this is ordinary load |
| Authorization codes ([ADR-005](005-account-sessions-and-rp-tokens.md) §2) | Same |
| Sessions | Already in the database (hash only) |
| Signing keys, VSID secret | Already in the database, encrypted; nodes reload on an unknown `kid` |
| Rate limits (`tower-governor`, in memory, per client address) | Kept per node as a floor, and a global limit at the ingress. With N nodes a client gets up to N times the quota: acceptable, since the limits protect resources, not secrets (a recovery code has 128 bits; a passkey cannot be guessed) |
| Master key | Given to every node by the platform's secret manager |

Rejected: putting the ceremony state in an encrypted cookie (single use cannot be enforced
without server state); Redis or Valkey (one more stateful service to run for something
PostgreSQL does, until a measurement says otherwise).

PostgreSQL's own availability is the operator's (an operator, a managed service). Several
regions are out of scope.

For `core`: `brigid-store` gets a second backend behind the same interface (sqlx supports
both; SQL and migrations are per backend), and the two in-memory maps move behind a trait
with a memory implementation and a database one.

## 4. Numbers

The forest line is inconsistent (a target of 1,000 and a test at 1,000,000), and the grove
and forest targets would put a 150-person company on a cluster. Proposal: one rule, **each
product is tested at ten times the top of its intended range**.

| | Intended accounts | Tested maximum | Identities at the maximum | Peak load to hold (§2) |
| --- | --- | --- | --- | --- |
| leaf | 1–10 | 100 | 1,000 | 0.3 per second |
| grove | 10–1,000 | 10,000 | 100,000 | 30 per second |
| forest | 1,000–100,000 | 1,000,000 | 10,000,000 | 2,900 per second |

- grove's range grows to 1,000: associations of several hundred members are common, and a
  free public server must take thousands without being a cluster.
- A size chooses neither the product nor the price by itself: a company of 300 that accepts
  a short outage runs a grove; a non-profit that cannot runs a forest. What is paid is in
  [ADR-020](020-licensing-and-pricing.md).
- **Limits are soft**: a warning in the administration and in the log beyond the tested
  maximum, never a refusal. A hard limit in free software can be removed by anyone, and a
  refusal at account 101 is, for the person it hits, a bug.

Load tests fill a server to its maximum, then hold the peak for an hour with a 95th
percentile under 300 ms for a sign-in, on a stated reference machine per product.

## 5. The names

Keep **leaf, grove, forest**. Nobody needs the order explained, which is rare.

- **forest**: in identity management, a "forest" is the top-level container of Microsoft
  Active Directory. An enterprise administrator will hear it. It is a resonance more than a
  conflict (it is their word for "the whole organisation"), but documentation must never
  use "forest" as a common noun. *Forest Admin* is an unrelated software product.
- **grove**: several "GROVE" software trademarks exist (project management, hardware
  modules), none found in identity management.
- **leaf**: "leaf certificate" in PKI; no conflict found.

They are edition names, always used with the brand ("brig·id grove"), never as marks of
their own. This is a quick look, not a clearance search: the name that deserves one, before
any registration, is **brig·id** itself.

## 6. Moving between products

An export/import format (accounts, passkeys, identities, the VSID secret, the signing keys;
encrypted, re-encrypted under the target's master key). VSIDs survive, since they do not
depend on the host ([ADR-007](007-identifiers.md)).

**The server's domain cannot change in a move**: a passkey is bound to the domain that
created it. Moving to another domain means every account enrolling new passkeys.

## Consequences

- `core`: storage and ceremony-state interfaces with two implementations; the export
  format. None of it is needed for v1 (`leaf`).
- `server-grove`: its README is corrected (SQLite, one container) when this is accepted.
- The site's editions page: the "How it is run" row and the volumes.

## References

- SQLite testing: <https://www.sqlite.org/testing.html>; redb: <https://www.redb.org>
- Active Directory forests:
  <https://learn.microsoft.com/windows-server/identity/ad-ds/plan/forest-design-models>
- [ADR-013](013-leaf-single-executable.md), [ADR-014](014-three-products.md),
  [ADR-020](020-licensing-and-pricing.md)
