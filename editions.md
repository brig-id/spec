# Editions, v1 and licence — brig·id

> **Audience:** auditors, operators and contributors.
> What v1 is, what the three editions are made of, and under which terms the software is
> given. Statuses are explained in the [README](README.md).

---

## 1. v1 is three steps

**Status:** decided.

v1 is not one release. It is reached in three steps, in this order, each usable for what it
is for.

| Step | It is reached when | What it contains |
| --- | --- | --- |
| **1. I use it** | The project's author signs in to a real application through the project's own server, every day, with the new interface | The sign-in of applications; the self-contained interface in its dark theme, in English and French; the demonstration site online; the new name of the smallest edition and the licence; automated releases |
| **2. Anyone can install it** | Someone who never spoke to the project installs a server and connects an application to it | seedling as one executable; **grove available**; administration; attributes, shown and accepted; the life of an account complete; the light theme; accessibility checked in CI; the design of end-to-end encryption; installation documentation |
| **3. It can be audited** | The scope is frozen, specified and tested | End-to-end encryption built; post-quantum signatures on request; these documents re-read against the code; load tests; **forest designed**; feature freeze; the audit |

v1 ships two products, seedling and grove, and the design of the third.

### Where it stands

- **Step 1: reached on 15 November 2026.** The project's own server runs seedling 0.4, and
  the demonstration site ([demo.brig.id](https://demo.brig.id)) signs in through it, as an
  application with a server and as one that runs entirely in the browser, with the new
  interface. One part of what the step lists is not complete: releases are automated for
  the interface only, and the update pull requests still wait for their tool to be
  installed ([`operations.md`](operations.md) §6).
- **Step 2: started.** The light theme and the accessibility checks in CI are built
  ([`interface.md`](interface.md) §3). So are attributes, shown and accepted
  ([`model.md`](model.md) §3–§5), the life of an account, from the identities proposed at
  onboarding to export and deletion ([`model.md`](model.md) §6), and the interface inside
  the executable (§3): seedling 0.5.0. Administration is built too
  ([`model.md`](model.md) §7): the first administrator from a setup link, accounts,
  applications, the sign-up policy and invitations, a log. Still to build: the rest of the
  single executable (TLS, `init`, static binaries, the installation guide), grove, backup
  and restore to move between seedling and grove (§5), the limits of a server open to
  others (§2), and the design of end-to-end encryption.

---

## 2. Three editions, one model

**Status:** decided. Today one server exists: seedling.

seedling, grove and forest are three products, each with its repository, its releases and
its documentation. An operator chooses one; going from one to another is a migration, not a
setting (§5).

**What an account, an identity, a passkey, a session and a token are does not depend on the
edition.** Protocol, cryptography and the security model live in the shared libraries
(`crypto`, `core`) and in the interface (`app`). No edition weakens the model, and a
security fix reaches the three. A product repository contains only what differs: packaging,
storage, listeners, clustering, operations.

Volume does not separate the editions. An identity server answers a few operations per
person and per day: with 25 operations per account and per day and a peak at ten times the
average, 10,000 accounts are 30 operations per second at peak, which an embedded database
on a small machine handles with a wide margin. **The editions differ by who runs them and by
what an outage costs.**

| | seedling | grove | forest |
| --- | --- | --- | --- |
| For | Oneself, a family, a test | A community, an association, a public server | A company |
| Run by | One person | A volunteer or a small team | A platform team |
| Deliverable | One executable | One container image, a compose file | Stateless containers, a chart, an integration guide |
| Needs | Nothing but itself | A container runtime | A cluster and a database |
| Storage | SQLite file | SQLite file on its own volume, **continuously replicated** to a second place | PostgreSQL, provided by the operator ([`forest.md`](forest.md) §3) |
| Availability | Stops with the machine; restored from a backup | One node; restarts in seconds; documented recovery point and time | No single point of failure; rolling updates without interruption |
| TLS | Built in | Built in, or a proxy | The cluster's ingress |
| Master key | A file or the environment | A container secret | The platform's secret manager |
| Sign-up by default | By invitation | Open or by invitation, with the limits below | By policy |
| Updates | Replace the file | Follow the image tags | Rolling, migrations as a job |
| Engine | **single node** | **single node** | **distributed** |

- **seedling and grove run the same engine and have the same functions**, administration
  included. What separates them is mainly that grove depends on a container runtime and
  seedling on nothing; then operations (continuous replication, metrics) and defaults. grove
  uses no PostgreSQL and no Redis: for one machine and a thousand accounts they would be two
  more services to run, back up and upgrade, for nothing.
- **grove starts from today's deployment** (the image, the compose file, the proxy), which is
  closer to it than to the single executable that seedling becomes.
- **SQLite stays.** It is already a library inside the binary and one file. A pure-Rust
  key-value store would save a C dependency at the price of a second implementation of the
  encrypted store, the layer where a divergence is a security bug.
- **One operator command line, the same in the three servers.** In v1 its commands run on
  the machine (inside the container for grove), on the database. A client that talks to a
  server from elsewhere needs a way to authenticate a machine, a kind of secret the product
  does not have: after v1, with its own design.
- **One configuration prefix** for the three (`BRIGID_…`).

### Sizes

Internal rule: **each edition is tested at ten times the top of its intended range.**

| | Intended accounts | Tested maximum | Identities at the maximum | Peak to hold |
| --- | --- | --- | --- | --- |
| seedling | 1–10 | 100 | 1,000 | 0.3 per second |
| grove | 10–1,000 | 10,000 | 100,000 | 30 per second |
| forest | 1,000–100,000 | 1,000,000 | 10,000,000 | 2,900 per second |

A load test fills a server to its maximum, then holds the peak for an hour with a sign-in at
the 95th percentile under 300 ms, on a stated machine.

- **The public site shows no ranges and no test figures.** It names the situation each
  edition is for (oneself or a family, a community, a company), with at most one example of
  size each.
- **No ceiling in the software.** Beyond its tested maximum a server warns in the
  administration and in the log, and refuses nothing. A hard limit in free software can be
  removed by anyone, and a refusal at account 101 is, for the person it hits, a bug. The
  limits below bound one account and one address, not the size of a server.

### Limits of a server open to others

**Status:** decided — step 2, in progress (core#94). Described in [`model.md`](model.md) §7.

A server whose sign-up is open, as a grove often is, stores what people it does not know
give it. Three limits bound that, the same in every edition, set by an administrator from
the interface (the page "Limits") or by the operator from the machine (the `limits`
command, [`operations.md`](operations.md) §7):

| Limit | Default | Maximum, in the code |
| --- | --- | --- |
| `identities_per_account` | 20 | 200 |
| `passkeys_per_account` | 10 | 50 |
| `sign_ups_per_ip_per_day` | 5 | 1,000 |

The defaults are the same for seedling and grove today. **Recommended values per edition**
will come from measuring sign-in and sign-up on small machines, against the sizes above;
the administration page will show them next to each limit. Until that measurement exists,
no edition has values of its own.

---

## 3. seedling: one executable

**Status:** built, except the `acme` mode in seedling itself (*in review*: core#122 builds it
in the engine, then seedling exposes it). Static binaries for Linux x86-64 and arm64 with
the interface inside (server-seedling#74, #77), `init`, the setup link and the systemd unit
(server-seedling#76), the sealed key.

One executable per target (Linux x86-64 and arm64 first), containing the interface (embedded
at compile time), the database engine, TLS and an ACME client. It needs no container
runtime, no reverse proxy and no database server: a data directory and a master key are all
it has outside itself. A container image remains as another packaging of the same binary.

| TLS mode | For |
| --- | --- |
| `acme` | A public name: certificates obtained and renewed from an RFC 8555 authority (Let's Encrypt by default, any directory otherwise), by TLS-ALPN-01 on port 443, renewed once two thirds of their lifetime have passed. The ACME account key and the certificate keys are stored encrypted in the database, like the other secrets ([`security-model.md`](security-model.md) §4.1) |
| `files` | A certificate and key provided by the operator |
| `off` | Plain HTTP behind a proxy that terminates TLS |

The built-in listener speaks TLS 1.3 only, in both modes.

- The process never runs as root: ports 443 and 80 are bound with `CAP_NET_BIND_SERVICE` or
  by socket activation; a systemd unit is shipped.
- **First start:** an `init` command creates the data directory and a configuration, and the
  server prints the setup link of the first administrator ([`model.md`](model.md) §7). For
  the master key, three documented choices: a key file created by `init`, which protects
  copies of the database and not the theft of the whole disk; the key supplied from
  outside the data disk; or the key sealed by the machine (`init --seal`, `seal-key`:
  `systemd-creds` with the TPM when there is one, the key shown once to keep offline),
  which protects the disk taken away without a second disk ([`security-model.md`](security-model.md) §9).
- **A static binary.** The server links to the system's OpenSSL only because the WebAuthn
  library depends on it to verify attestation certificates, which brig·id never asks for.
  The release build is a static binary with OpenSSL vendored.
- **No self-update:** a server that replaces itself from the network is an attack surface.
- A release build without the interface, or without its list of script hashes, fails: no
  fallback to a weaker Content Security Policy ([`security-model.md`](security-model.md) §4.6).
- The interface is embedded with its text files compressed (about 0.7 MB for the whole of
  it, against a limit of 2 MB that the build enforces). **A page is a file of the build**:
  a path that names nothing answers `404`, and no page answers for the others. Files named
  after their content are cached for good, pages are checked at each visit.
- A directory holding another build of the interface can be served instead, as a
  development option (`server.ui_dist_dir`), through the same code.

---

## 4. forest: designed, not built

**Status:** decided — step 3, as a design: [`forest.md`](forest.md). Not built.

forest runs several nodes that hold no state, on a PostgreSQL database the operator
provides. Its design has its own document: what leaves a node's memory, where it goes and
on which measurements, how a server starts, is updated and fails, what is delivered.

Two options were considered and rejected for the state nodes must share. In an encrypted
cookie: single use cannot be enforced without server state. In a cache next to the
database: one more service to run and to secure, for a load the database holds
([`forest.md`](forest.md) §3).

---

## 5. Moving between editions

**Status:** decided — step 2 for seedling and grove, in progress (core#94); decided — after
v1 for forest.

**The server's domain cannot change in a move**: a passkey is bound to the domain that
created it. Identifiers at applications survive: they depend on the VSID secret and on the
identity, not on the host ([`protocol.md`](protocol.md) §2). The project's own server
follows this path on its domain: it becomes a grove when grove exists, and a forest when
forest does.

### Between seedling and grove: a backup

seedling and grove run the same engine on the same SQLite database (§2), so a move between
them is a backup of one restored into the other. Two operator commands, the same in both
([`operations.md`](operations.md) §7):

- `backup` writes a **consistent copy of the database while the server runs** (SQLite's
  online backup). The copy is encrypted as the database is, under the **same master key**:
  it holds everything the server needs (accounts, passkeys, identities, the VSID secret,
  the signing keys), and the master key is not in it.
- `restore` puts a backup in place of the database of a server that is stopped. The target
  starts with the same master key and on the same domain. It runs the same version as the
  source or a newer one: it upgrades the schema it finds, and refuses a database whose
  schema is newer than it knows.

Nothing is re-encrypted and nothing is converted: the target reads the file as the source
wrote it. The same two commands are how any seedling or grove is backed up and restored
when nothing moves.

### To forest: an export

forest stores its data in PostgreSQL ([`forest.md`](forest.md) §3), so a SQLite file cannot
be restored into it. Coming from grove needs an export and an import of the server's data
(accounts, passkeys, identities, the VSID secret, the signing keys), encrypted, re-encrypted
under the target's master key at import. *Decided — after v1*: it is needed only once forest
is built.

---

## 6. Names

**Status:** built.

| Edition | Name | Was |
| --- | --- | --- |
| Smallest | **seedling** | leaf |
| Middle | **grove** | grove |
| Largest | **forest** | forest |

One picture: something that grows into a wood. A seedling's leaves are its users. In
identity management a "forest" is also the top-level container of Microsoft Active
Directory; it names something close, which is acceptable, and these documents never use the
word as a common noun. They are edition names, always used with the brand, never as marks of
their own.

The repository, the binary, the published image and these documents say `seedling`; the
three editions read their settings under one prefix, `BRIGID_…`. "leaf" may later name any
server of a brig·id network ([`model.md`](model.md) §8); nothing is decided on that. The
project's own machine was named `leaf-1` before the change and keeps that name: it names a
machine, not the edition.

---

## 7. Licence, and what is paid

**Status:** built for the licences. The contribution rule and the trademark policy are
*in review* (brig-id/.github, `CONTRIBUTING.md` and `TRADEMARKS.md`); filing the mark is
decided — step 2 and not done. Versions released before the change keep the licence they
were published under.

- **The servers, the shared core and the interface: AGPL-3.0-or-later.** One licence for the
  three servers, all public. The core follows, or the servers' licence would protect
  nothing: almost all the logic is there, and a library under the LGPL may be linked into a
  closed program.
- **The cryptography library stays LGPL-3.0-or-later**: a thin library over public
  primitives gains from being reused and reviewed.
- **No commercial licence, ever.** The project never sells an exception to the AGPL. Nobody,
  its author included, can take the code closed: for a product that says "you do not have to
  trust us", this is the strongest form of the promise. What is given up: the organisations
  that forbid AGPL software and would have bought other terms.
- **Contributions under a Developer Certificate of Origin** (version 1.1), nothing to sign.
  Each commit carries a `Signed-off-by` line with its author's name and address
  (`git commit -s`), which certifies the contributor may give the change under the
  repository's licence. With no other licence to grant, no contributor agreement is needed.
  A check on pull requests verifies the line on every commit, for people outside the
  organisation; its members and the update bots are exempt.
- **The name is to be registered** as a trademark, subject to a clearance search: a first
  look found earlier marks close to it, and the filing waits for professional advice. It is
  not registered today. A short policy says what others may do with the name: a fork or a
  host may say truthfully that it is based on brig·id or compatible with it; it may not call
  itself brig·id or suggest it is the project. A trademark does not stop anyone from running
  the software for free, nor a fork under another name; with the AGPL, which makes whoever
  offers a modified version as a service publish the changes, it is the realistic protection.

**"AGPL, with commercial use forbidden" does not exist.** The AGPL grants everyone the right
to run the program for any purpose, and lets a recipient remove a restriction added to it.
A licence that makes companies sign before using the software is not free software, which
would also close the funded audit ([`audit/`](audit/README.md)).

**What is paid, for any edition:** support and guaranteed updates; hosting by the project;
and **modules that are separate programs**, talking to a server through its interfaces (a
replacement interface, white-label). A paid module cannot be a part linked into the server:
the server contains contributors' code, which the project has no right to close.

**No security feature is ever paid.** Everything that touches authentication, identities,
tokens and cryptography is free software and in the audit's scope. Nothing is gated inside
the code, and no edition has a ceiling.
