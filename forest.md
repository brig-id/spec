# forest — brig·id

> **Audience:** auditors, the platform teams that will run it, and contributors.
> The design of the largest edition: what it is made of, where its state lives, how it
> starts, is updated and fails. What the three editions are, and what separates them, is in
> [`editions.md`](editions.md) §2. Statuses are explained in the [README](README.md).

Nothing in this document is built. A section marked *decided* is a commitment about the
design; a section marked *open* says what is not settled, and why.

---

## 1. What forest is

**Status:** decided — step 3, as a design. Not built.

forest is a brig·id server made of several identical **nodes**. A node holds nothing that
another node needs: any node answers any request, a node can stop in the middle of a
sign-in without that sign-in being lost, and an update replaces the nodes one by one.

```text
        browsers, applications
                 │  HTTPS
        ┌────────▼────────┐
        │     ingress     │   the cluster's: TLS, the global rate limit
        └──┬─────┬─────┬──┘
           │     │     │      plain HTTP inside the cluster
        ┌──▼─┐ ┌─▼──┐ ┌▼───┐
        │node│ │node│ │node│  the same image, no disk, no state
        └──┬─┘ └─┬──┘ └┬───┘
           └─────┼─────┘      TLS, the server certificate verified
          ┌──────▼───────┐
          │  PostgreSQL  │    the company's: everything that lasts,
          └──────────────┘    and everything the nodes share
```

- **One server is one domain and one database.** forest is not a way to link servers:
  accounts and identities are never linked across servers ([`model.md`](model.md) §8).
- **The same engine.** A node runs the code seedling and grove run: the same routes, the
  same ceremonies, the same tokens, the same interface, the same operator commands. What
  an account, an identity, a passkey, a session and a token are does not change. Two
  things differ, and only these: where state lives (§2, §3), and how a server starts and
  is updated (§5, §6).
- **The company provides what it already runs:** a cluster, an ingress that terminates
  TLS, a PostgreSQL database, a secret manager. forest brings none of them and embeds no
  client for any of them: it reads files and speaks PostgreSQL.
- **What forest promises is that no node is a single point of failure.** The database is
  one: when PostgreSQL does not answer, nobody signs in. Its availability (a standby, a
  failover, backups) is the operator's, with the tools they already trust for it. forest
  says what it needs of the database (§3) and how it behaves while the database fails over
  (§7).

---

## 2. What leaves a node's memory

**Status:** decided — step 3, as a design. Not built: today every line of the first column
is true of the single node that seedling and grove run.

| State | In a single node today | In forest |
| --- | --- | --- |
| Passkey ceremonies in progress (registration, sign-in, confirmation) | In memory, 5 minutes | In the database (§3): shared, with an expiry, taken once whichever node answers |
| Sign-in requests of applications waiting for the person | In memory, 10 minutes | Same |
| Authorization codes | In memory, 1 minute | Same |
| Assertions of clients already presented (`private_key_jwt`, [`protocol.md`](protocol.md) §6) | In memory, until they expire | In the database, in a table that survives a crash: an assertion is accepted once by the server, not once per node |
| Sessions | In the database (a hash only) | Unchanged |
| Accounts, passkeys, identities, what was accepted, applications, settings, links, the log | In the database | Unchanged |
| Signing keys, the VSID secret | In the database, encrypted; read at start | Unchanged; created once by the initialisation (§5); a node learns of a rotation without restarting (§6) |
| Documents and keys of applications that describe themselves, read lately | In memory | In memory, per node: it is a cache. The last valid document is in the database, as today |
| Rate limits per client address | In memory | In memory per node, as a floor; the limit that counts is the ingress's (§4) |
| The server's own certificate (ACME) | In the database | None: forest has no TLS listener |
| Metrics | In memory | In memory per node; each node is scraped |
| Master key | A file, the environment, or sealed by the machine | A file mounted in every node by the platform's secret manager (§4) |

Two lines deserve attention because nothing fails when they are wrong: the assertions of
clients, and what a first start creates (§5). In both, two nodes can disagree without any
error being raised.

---

## 3. Where the state goes: PostgreSQL, and nothing else

**Status:** decided — step 3, as a design, on the measurements below. Not built.

Everything a node shares with the others is in one PostgreSQL database. There is no cache
and no queue next to it.

### The store is not written twice

The encrypted store ([`security-model.md`](security-model.md) §4.5) is the layer where a
divergence between two implementations is a security bug: the reason SQLite stays in the
two other editions ([`editions.md`](editions.md) §2). forest does not add a second store.
**The same statements run on both databases**:

- The tables have the same shapes in both: identifiers as text, times as Unix seconds in
  64-bit integers, encrypted values as bytes. No type exists in one database only.
- A statement is written once. The few places where the two databases need different text
  (the schema itself; the smaller of two values) are a named list in one file, and the
  list is short enough to be read in a review.
- What encrypts, derives a key, builds a lookup index or decides is above the statements
  and has one implementation.
- **Every test of the store runs on both databases in CI.** A release of `core` in which
  a test passes on one and fails on the other does not exist.

### The state shared between nodes

One table, `pending`, holds what §2 moves out of memory: a kind, an identifier, a value,
an expiry.

- **Taken once.** Finishing a ceremony, answering a request and exchanging a code each
  remove the row and return it in a single statement (`DELETE … RETURNING`). Of two nodes
  that present the same identifier at the same instant, the database gives the row to
  one; the other finds nothing and refuses. This is the property the option of an
  encrypted cookie could not give ([`editions.md`](editions.md) §4).
- **Nothing readable.** The value is encrypted like every sensitive field, under a key
  derived from the master key: it carries what a person accepted to share, and the state
  of a passkey challenge. The identifier stored is the SHA3-256 hash of the one the
  browser or the application holds, as for sessions: reading the table gives nothing that
  can be presented.
- **Expired rows** are removed by every node every few seconds. Two nodes doing it at once
  is harmless.
- **`pending` is an unlogged table.** Its rows live minutes and are worth nothing after a
  crash. An unlogged table is not written to the database's log: it costs less, and after
  a crash or a failover it comes back **empty**. The failure is then closed: ceremonies in
  progress start again, and no code or challenge already used can come back. The
  alternative that makes the same saving, a logged table with asynchronous commit, fails
  open: a crash can lose the last removals, and a code taken a moment before could be
  taken again.
- **The assertions of clients already presented** are in a second, ordinary table. Their
  whole purpose is to be remembered: emptied by a crash, it would accept again the
  assertions of the last minutes.

### What was measured

The statements a request costs in `core`'s store were played against PostgreSQL 18 filled
to forest's tested maximum ([`editions.md`](editions.md) §2): 1,000,000 accounts,
10,000,000 identities, 300,000 open sessions, 8.8 GB. The load is the peak to hold, 2,900
requests per second: 150 sign-ins to an application per second with a passkey ceremony
(six requests each), 350 from an open session (four requests), 120 visits to the account's
pages (five requests). One sign-in in ten is abandoned. Requests arrive on time whether or
not the earlier ones were answered, so a database that falls behind is seen. The tool and
its instructions are in the forest repository (`bench/`).

Only the database is measured: what a node computes (the passkey signature check, the
token signature, encryption) is the same in every option and scales with the number of
nodes.

**The machine.** PostgreSQL 18.7 in a container held to 4 processors and 8 GB of memory
(`shared_buffers` 1 GB, `max_wal_size` 8 GB, `checkpoint_timeout` 15 minutes; `fsync` and
`synchronous_commit` on), on a laptop (AMD Ryzen 5 6600H, NVMe disk); the tool on the same
machine, 64 connections; the cache is Valkey 8.1 without persistence. Each run measures
120 seconds after 30 seconds of warm-up. Three runs for each option.

Time a whole sign-in waited for the database, all its requests added, in milliseconds:

| Where the shared state lives | With a passkey (6 requests): median | 95th percentile, best and worst run | From a session (4 requests): median | 95th percentile, best and worst run |
| --- | --- | --- | --- | --- |
| A table | 52 to 56 | 62 to 463 | 35 to 38 | 43 to 358 |
| **An unlogged table** | **30 to 36** | **35 to 81** | **20 to 24** | **24 to 58** |
| A cache next to the database | 29 to 31 | 73 to 529 | 18 to 20 | 51 to 406 |

- **PostgreSQL alone holds the peak, with room.** A sign-in waits about 30 ms for the
  database, a tenth of the 300 ms a sign-in may take in all. At twice the peak (5,800
  requests per second) the medians do not move (56, 30 and 25 ms) and the database uses
  under three of its four processors.
- **A cache buys nothing.** It answers as fast as the unlogged table at the median and
  worse beyond, and it removes none of what costs: the writes that must last (sessions,
  choices, what was accepted) stay in PostgreSQL. It would be a second service to run,
  to secure and to keep available, and the assertions of clients would need a durable
  place anyway.
- **An unlogged table costs a third less than an ordinary one**, and is the steadiest of
  the three.
- **The tails are this machine's.** Its disk stalls at times, and the stalls show in the
  95th percentile of some runs of every option. The medians and the order of the options
  are stable from run to run; percentiles to publish come from the edition's load test
  (§10), on a stated database.
- **The database needs four processors for the tested maximum.** Held to 2 processors and
  4 GB, it does not hold the peak in any option (its processors are the limit, and
  sign-ins wait seconds); it holds half of it with the same times as above. The intended
  range stops at a tenth of the tested maximum.
- **Writing a session's deadline less often was measured and not adopted.** Today every
  request pushes the idle deadline back ([`protocol.md`](protocol.md) §5). Writing it only
  when it moves by a minute saves 4 to 7 ms on a sign-in: not enough to make the three
  editions behave differently from what the protocol says, and not needed to hold the
  peak.
- At the peak the database writes 5 to 7 MB of log per second, mostly whole pages: a
  figure for whoever sizes the archive of the log.
- **What the medians are made of.** Each write that must last waits once for the disk,
  about 4 ms on this machine, and a sign-in makes several: the same times appear at a
  tenth of the load. The statements were played as `core` issues them today, one by one.
  Grouping the writes of a request in one transaction was not measured; it can only
  lower these figures, and it is how the store will write on PostgreSQL, so that a
  request's writes happen together or not at all.

### What PostgreSQL must be

- A version the PostgreSQL project still supports. CI runs the store's tests on the oldest
  and the newest supported versions.
- Reached over TLS with the server's certificate verified; a node refuses to start on a
  connection without TLS unless the operator wrote that choice in the configuration.
- One role for the nodes, which cannot change the schema; another for the initialisation
  (§5), which can.
- A primary that accepts writes. forest does not read from standbys: a session written a
  moment ago must be found by the next request.
- Unlogged tables available. Where a managed service does not offer them, `pending` is an
  ordinary table (a setting of the initialisation): measured above, it holds the peak too,
  at a higher cost for the database.

---

## 4. Around the nodes

**Status:** decided — step 3, as a design. Not built.

- **TLS** ends at the cluster's ingress. A node listens in plain HTTP inside the cluster
  and has no certificate and no ACME client. The ingress must **replace** the
  `X-Forwarded-For` header, never append to it: nodes key their rate limits on it.
- **Rate limits.** Each node keeps the limits of the single-node server, in memory, per
  client address: a floor that needs nothing shared. Behind an ingress that spreads
  requests, a client can reach each node's limit; the limit that counts is therefore
  **the ingress's**, global, and the integration guide gives its values. Counters shared
  in the database were rejected: one more write per request, for a job the ingress
  already does.
- **The cap on ceremonies in progress.** A single node refuses new ceremonies beyond
  10,000 held in memory. In forest the count is read from the database by each node every
  few seconds, and new ceremonies are refused while it is over the cap: a flood fills a
  table up to a bound, not the database.
- **The master key** is a file mounted in every node by the platform (a Kubernetes secret,
  an external-secrets operator, a CSI driver, an agent's sidecar): `BRIGID_MASTER_KEY_FILE`,
  as in grove. The database's credentials arrive the same way. forest contains no client
  for any secret manager: nothing more to audit, and nothing to choose.
- **Probes.** Liveness: the process answers. Readiness: the database answers, the schema
  is the one this version expects (§6), the keys are loaded. A node that receives the
  signal to stop fails its readiness first, then finishes the requests it has.
- **Clocks.** Expiry times are written and compared by nodes. Nodes must be synchronised
  to within a few seconds, which the validity of tokens already asks.
- **Operator commands** are the same as in the other editions and act on the database.
  They are run in a node's container or as a job of the cluster; the nodes see their
  effect at the next request, or at their next reading of the keys (§6).

---

## 5. First start

**Status:** decided — step 3, as a design. Not built.

A single-node server creates what is missing when it starts: the schema, the VSID secret,
the first signing key, the setup link of the first administrator. Several nodes starting
together must not each do it: two VSID secrets would give two sets of identifiers to
applications, and each new setup link cancels the one before.

In forest **nodes create nothing**. One command, `forest init`, run once as a job before
the nodes start:

1. takes a lock in the database, so that two of them cannot run together;
2. creates or migrates the schema (§6);
3. creates the VSID secret and the first signing key if they do not exist;
4. prints the setup link of the first administrator if the server has none
   ([`model.md`](model.md) §7).

A node that finds no schema, a schema it does not know, no VSID secret or no signing key
is not ready and says why. It never repairs.

---

## 6. Updates and rotations

**Status:** decided — step 3, as a design, except the rotation of the master key, which is
**open**. Not built.

### Rolling updates

An update replaces the nodes one by one: for a while, nodes of two versions answer on one
database. Hence one rule for every change of the schema: **a migration must leave the
previous version working.** A column is added in one release and relied on in the next; it
is removed one release after the code stopped reading it. A release's notes say when an
update must not skip a version.

- Migrations are run by `forest init`, as a job before the first new node starts, never by
  a node.
- A node knows the oldest and the newest schema it can work with. Outside that range it is
  not ready: an old node left behind stops receiving requests instead of writing wrongly.
- The short-lived state needs no migration: a new shape gets a new kind, and rows of the
  old kind expire within minutes.

### Signing keys

Today a rotation is a command and a restart ([`operations.md`](operations.md) §2). In
forest a rotation needs no restart, and a token must never be signed with a key that some
node does not publish yet:

- every node reads the signing keys from the database again every minute;
- a new key is **published first, used later**: it appears in every node's key set as soon
  as the node has read it, and nodes sign with it only once it is five minutes old;
- the key it replaces stays published for its grace period, as today.

No node talks to another, and no order has to be followed between them.

### The master key

**Status:** open.

Rotating the master key re-encrypts every row, with the server stopped
([`operations.md`](operations.md) §1). On forest that is an interruption, the one thing the
edition exists to avoid. A rotation without interruption needs nodes that hold the old key
and the new one for a time, read with either and write with the new one, while a job
re-encrypts the rows. It touches the encrypted store, for the three editions, and is
designed with it. Until then a rotation of the master key in forest is a maintenance
window, as in the other editions.

---

## 7. What fails, and what happens

**Status:** decided — step 3, as a design. Not built.

| Failure | Effect |
| --- | --- |
| A node stops, crashes or is replaced | Nothing is lost: its requests in flight fail and are retried by the browser or the application; ceremonies and requests in progress continue on another node |
| A node cannot reach the database | It is not ready and receives nothing |
| The database fails over to a standby | Nobody signs in until it answers. Sessions survive. Ceremonies and sign-ins in progress start again: `pending` comes back empty (§3) |
| The database is restored from a backup | Sessions opened and passkeys added since the backup are gone, as in any edition |
| Two nodes present the same code or finish the same ceremony | One succeeds (§3) |
| A node's clock is wrong | Within seconds: nothing. Beyond: it can refuse a ceremony still valid or accept one just expired. Synchronised clocks are a requirement (§4) |
| An update is half done | Two versions answer, by the rule of §6 |

---

## 8. What is delivered

**Status:** decided — step 3, as a design. Not built.

- **An image**: the server as one static executable in a minimal image, like grove's, with
  the interface inside. It runs as a user without privileges and writes nothing to disk.
- **A Helm chart**: the nodes (two at least, spread over the cluster's machines, with a
  disruption budget), the initialisation job before each installation and update, the
  service, and as options the route through the ingress, a network policy that lets nodes
  reach the database and nothing else, and what a metrics collector needs. The chart
  creates no database and no secret.
- **An integration guide**: what to provide (the database and its two roles, the key, the
  name), the ingress's settings (TLS, the header, the global rate limit and its values),
  sizing from the measurements of §3, updates, what to watch.
- A cluster is not required by the server itself: a node is a container that needs a
  database and a file. The chart is what the project tests.

Kubernetes is the one platform the project packages and tests for.

---

## 9. Coming from grove

**Status:** decided — step 3, as a design; the export and import it needs are decided —
after v1 ([`editions.md`](editions.md) §5).

A grove becomes a forest by an export and an import: a SQLite backup, which moves a
seedling to a grove, cannot be restored into PostgreSQL. The tables have the same shapes in
both databases (§3), so an import is a copy of rows, re-encrypted under the target's master
key; the domain does not change. The project's own server follows this path.

---

## 10. Sizes and the load test

**Status:** decided — step 3, as a design. The test itself comes with the build.

forest is intended for 1,000 to 100,000 accounts and tested at 1,000,000
([`editions.md`](editions.md) §2). The measurements of §3 are of the database alone and
decide the design. The load test of the edition is another thing: real nodes, real passkey
ceremonies and tokens, the peak held for an hour with a sign-in at the 95th percentile
under 300 ms, on a stated cluster and a stated database. Its figures replace those of §3
in the integration guide when it exists.
