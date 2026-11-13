# ADR-019 — Editions: what seedling, grove and forest are made of

**Status:** Accepted (2026-11-13, review round 3 with the owner). Follows
[ADR-014](014-three-products.md). **The smallest edition, `leaf` until now, is renamed
`seedling`** (§5); this record uses the new name.

## Context

The working table for the three products (the maximum is what load tests must reach; ten
identities per account on average):

| | Accounts (target, tested maximum) | Deliverable | Machine | For |
| --- | --- | --- | --- | --- |
| leaf (now seedling) | 1–10 (50) | one executable | a micro-server, one service among others | tests, a personal server |
| grove | 10–100 (1,000) | a container image or a set of containers | a dedicated machine | non-profits, associations, free public servers |
| forest | 100–1,000 (1,000,000) | a set of pods and how to assemble them | a cluster | companies, very large non-profits, resellers |

Five questions were asked. They are answered in order, with what the review decided.

## 1. Can the smallest edition be "simpler than SQLite"?

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

What can be simpler is what the operator touches, not the engine: no configuration to write
([ADR-013](013-leaf-single-executable.md)), a `backup` command producing one consistent file
while the server runs, and a clean stop leaving exactly one file.

**Decision: seedling keeps SQLite.**

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

| | seedling | grove | forest |
| --- | --- | --- | --- |
| Operated by | One person, for themself, a family, a test | A volunteer or a small team, for an organisation or the public | A platform team |
| Deliverable | One executable | One container image, a compose file | Stateless containers, a Helm chart, an integration guide |
| Needs | Nothing but itself | A container runtime | A cluster and a database |
| Storage | SQLite file | SQLite file on its own volume, **continuously replicated** to a second place | Decided with forest's design (§3) |
| Availability | Stops with the machine; restored from a backup | One node; restarts in seconds; documented recovery point and time | No single point of failure: two nodes or more, rolling updates without interruption |
| TLS | Built in (ACME) | Built in, or a proxy | The cluster's ingress |
| Master key | A file or the environment | A container secret | The platform's secret manager |
| Sign-up by default | By invitation | Open or by invitation: a public server needs quotas per account and abuse controls | By policy |
| Updates | Replace the file | Follow the image tags (what `helm.brig.id` does) | Rolling, migrations as a job |
| Engine | **single node** | **single node** | **distributed** |

**Decisions:**

- **`grove` runs on the same engine as `seedling`**: SQLite, one process, in a container
  image. No PostgreSQL and no Redis: for one dedicated machine and a thousand accounts they
  would be two more services to run, back up and upgrade, for nothing, and the person who
  runs a grove is often a volunteer.
- **`seedling` and `grove` have the same functions**, administration included. What
  separates them is mainly that grove depends on a container runtime and seedling on
  nothing; then operations (continuous replication, metrics) and defaults.
- **One operator command line, the same in the three servers.** In v1 its commands run on
  the machine (inside the container for grove), on the database, like today's key rotation
  and site registration. A client that talks to a server from elsewhere needs a way to
  authenticate a machine, a new kind of secret in a product that has none: its own record,
  after v1.
- **One configuration prefix** for the three (`BRIGID_…`), since they run the same code.

So: three products, **two engines**. `seedling` and `grove` share the single-node engine;
`forest` is the other.

## 3. forest: stateless nodes; where the state goes is decided with its design

forest runs several nodes that hold no state. What must leave the process memory:

| State today | Must become |
| --- | --- |
| WebAuthn ceremonies in progress (in memory) | Shared between nodes, with an expiry, and single use guaranteed whichever node answers |
| Authorization requests and codes of relying parties (in memory) | Same |
| Sessions | Already in the database (hash only) |
| Signing keys, VSID secret | Already in the database, encrypted; nodes must learn of a rotation |
| Rate limits (in memory, per client address) | Per node as a floor, and a global limit somewhere in front |
| Master key | Given to every node by the platform's secret manager |

**Not decided now.** Where this state goes, and on which database forest stands, is decided
at step 3 of v1, with forest's design record, on measurements: speed, security,
compatibility with what companies already run, and operating cost. The working hypothesis
to test is PostgreSQL alone (temporary state in tables with an expiry, a global rate limit
at the ingress); a shared cache next to it is the alternative. One option is already
rejected: ceremony state in an encrypted cookie, because single use cannot be enforced
without server state.

For `core`, whatever the answer: the store behind an interface with a second backend, and
the in-memory maps behind a trait.

## 4. Numbers

The original forest line was inconsistent (a target of 1,000 and a test at 1,000,000), and
the targets would have put a 150-person company on a cluster.

**Internal rule: each product is tested at ten times the top of its intended range.**

| | Intended accounts | Tested maximum | Identities at the maximum | Peak load to hold (§2) |
| --- | --- | --- | --- | --- |
| seedling | 1–10 | 100 | 1,000 | 0.3 per second |
| grove | 10–1,000 | 10,000 | 100,000 | 30 per second |
| forest | 1,000–100,000 | 1,000,000 | 10,000,000 | 2,900 per second |

**Publicly, no ranges and no test figures.** The site names the situation each edition is
for: a family or oneself, a community or an association, a company. At most one example of
size goes with each (5, 500, 50,000), and it is not required.

- A size chooses neither the product nor a price by itself: a company of 300 that accepts a
  short outage runs a grove; a non-profit that cannot runs a forest.
- **No ceiling in the software**: beyond its tested maximum a server warns in the
  administration and in the log, and refuses nothing. A hard limit in free software can be
  removed by anyone, and a refusal at account 101 is, for the person it hits, a bug.

Load tests fill a server to its maximum, then hold the peak for an hour with a 95th
percentile under 300 ms for a sign-in, on a stated reference machine per product.

## 5. The names: seedling, grove, forest

The three were renamed in the review, within one picture: something that grows into a wood.

| Edition | Name | Was |
| --- | --- | --- |
| Smallest | **seedling** | leaf |
| Middle | **grove** | grove |
| Largest | **forest** | forest |

- **seedling**: a young plant, and its leaves are its users. No clash found.
- **grove** and **forest** are kept: their order needs no explanation. In identity
  management a "forest" is also the top-level container of Microsoft Active Directory; it
  names something close (the whole organisation), which is acceptable. Documentation never
  uses "forest" as a common noun.
- Set aside because they collide with a tool close to ours: acorn, sapling, timber, sequoia,
  heartwood, arbor, canopy, orchard, taiga.
- **"leaf" is free again.** It may come back to name any server connected to the brig·id
  network, whatever its edition. Nothing is decided on that.

They are edition names, always used with the brand ("brig·id grove"), never as marks of
their own. This was a quick look, not a clearance search; the name to register is
**brig·id** ([ADR-020](020-licensing-and-pricing.md)).

**The rename happens now**, at step 1 of v1, while one server runs the software: the
repository, the binary, the published image and the configuration. Until it lands, the code
and the documents that describe it as built still say `leaf`.

## 6. Moving between products

An export/import format (accounts, passkeys, identities, the VSID secret, the signing keys;
encrypted, re-encrypted under the target's master key). VSIDs survive, since they do not
depend on the host ([ADR-007](007-identifiers.md)).

**The server's domain cannot change in a move**: a passkey is bound to the domain that
created it. Moving to another domain means every account enrolling new passkeys.

The project's own server, `helm.brig.id`, follows this path on its domain: it becomes a
grove when grove exists, and a forest when forest does.

## Consequences

- `server-leaf` becomes `server-seedling`: repository, binary, image and configuration
  prefix, coordinated with the deployment of `helm.brig.id`, which follows the published
  image by its name.
- `core`: the operator commands shared by the three servers; storage and ceremony-state
  interfaces (needed for forest, not before); the export format (needed before grove).
- `server-grove`: its README is corrected (SQLite, one container).
- The site's editions page: situations instead of volumes, the new name, "how it is run".

## References

- SQLite testing: <https://www.sqlite.org/testing.html>; redb: <https://www.redb.org>
- Active Directory forests:
  <https://learn.microsoft.com/windows-server/identity/ad-ds/plan/forest-design-models>
- [ADR-013](013-leaf-single-executable.md), [ADR-014](014-three-products.md),
  [ADR-020](020-licensing-and-pricing.md)
