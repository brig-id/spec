# Model — brig·id

> **Audience:** external security auditors and advanced contributors.
> What the things are: accounts, identities, what an application receives, the life of an
> account, administration, servers. How they are carried over the wire is in
> [`protocol.md`](protocol.md); what protects them, in [`security-model.md`](security-model.md).
> Statuses are explained in the [README](README.md).

---

## 1. Account

**Status:** built.

An account is a person's private way in. It holds passkeys and a recovery code, and nothing
that describes the person: no profile, no e-mail address, no password. It is never presented
to an application.

- **Passkeys are the only way to authenticate** ([`security-model.md`](security-model.md) §5).
  An account has at least one; the server refuses to delete the last.
- **A username exists and is hidden.** The server generates it at sign-up (readable and
  random, like `swift-otter-4821`). The person never types it: signing in uses the passkey
  alone. It is what a passkey manager shows next to the passkey. It is unique per server,
  and nothing outside the account depends on it: no identifier, no token, no recovery code.
  Renaming it must stay possible; the endpoint is not built.
- **The account is identified by a UUID**, never shown.
- **A passkey always opens an account**, never one identity alone. An earlier design kept the
  possibility of a passkey limited to one identity; it is closed.

*Implementation:* `core`, `brigid-api/src/routes/auth.rs`; `brigid-store` (tables `users`,
`webauthn_credentials`); username generation in `brigid-identity/src/username.rs`.

---

## 2. Identity

**Status:** built, except where a line says otherwise.

An identity is what a person presents to the outside. It belongs to exactly one account, and
an account has several. Two identities of one account cannot be told to be the same person
by what brig·id gives out ([`protocol.md`](protocol.md) §2).

| Part | What it is | Who sees it |
| --- | --- | --- |
| **Identifier** | A UUID, immutable. The VSID an application receives is derived from it | The server |
| **Label** | A name (required), an icon (required), a description (optional) | The person only. **No application ever receives a label** |
| **Attributes** | Information about the person, optional (§3) | An application, after acceptance (§5) |

- The **name** is a display name: not a way to sign in, and not unique.
- The **icon** is a reference into the icon library of the interface, stored as text:
  `icon:ph:<name>`, the name of an icon of the Phosphor set
  ([`interface.md`](interface.md) §4). The server checks its shape only.
- An identity has no username to sign in with, and no DID.
- **Templates** only pre-fill a label when an identity is created. An identity does not
  remember its template, and a template carries no security meaning.

### The links of an identity

**Status:** decided — step 2 for the page; applications exist with the sign-in flow.

An identity's links with the outside are sorted by direction. These are the two headings of
an identity's page, in English and in French:

| | **Used by** (*Utilisée par*) | **Connected to** (*Reliée à*) |
| --- | --- | --- |
| Holds | What uses the identity | What the identity uses |
| Categories so far | **Applications**: third-party websites, and the person's own applications. **Sessions**: where the identity is signed in | **Services**: an outside service the identity is connected to, for which the server holds an access or runs the service. **Providers**, and others, when a need appears |
| The server is | The provider: what uses the identity trusts it | The client, or the operator of the service |
| In v1 | Applications | Nothing (§9) |

The word "connector", used earlier for all of these, is dropped.

*Implementation:* `brigid-store` (table `identities`), `brigid-api/src/routes/identities.rs`.

---

## 3. Attributes

**Status:** built, except two lines marked *decided — step 2*: the
photo, and the confirmation of an address by a message. Stored so that end-to-end encryption
([`security-model.md`](security-model.md) §7) takes them over at step 3 without a second
migration: the attributes of an identity are one encrypted value, under the same key as its
label, and a copy of the database does not tell which attributes an identity has.

Attributes are values a person puts on an identity in order to share them. They are
optional and typed, and they belong to one identity: nothing is copied from one identity to
another, and the account has none.

| Attribute | OpenID Connect claim | Scope |
| --- | --- | --- |
| First name, last name | `given_name`, `family_name` | `profile` |
| A username chosen for this identity | `preferred_username` | `profile` |
| A photo or avatar. *Decided — step 2* | `picture` | `profile` |
| Language, time zone | `locale`, `zoneinfo` | `profile` |
| An e-mail address | `email`, `email_verified` | `email` |

- The username attribute is a handle shown to applications, not a name to sign in with.
- The server checks the shape of a value (an address looks like one, a language is a
  language tag, a time zone a zone name) and nothing else: it does not know whether a first
  name is the person's.
- **E-mail** is optional, and the interface recommends it, because many applications need
  one. *Built today:* it is always shared as unverified (`email_verified: false`).
  *Decided — step 2:* if the server's administrator configured outgoing mail, the address is
  confirmed by a message and shared as verified. A small server is not required to send
  mail. A made-up, undeliverable address is never generated: it would be a false statement
  to the application.

**Attributes can link identities.** The per-application identifier keeps two identities
apart; the same last name, username or address given through two identities undoes that,
and no protocol can prevent it. The interface says so where an attribute is filled in, and
warns clearly when an address is already used on another identity of the account (a variant
of the same mailbox, `name+work@…`, protects little). It lets the person decide.

*Implementation:* `core`, `brigid-identity/src/attributes.rs` (the list, the shapes);
`brigid-store` (column `identities.attributes`); the interface's page of one identity,
`/identities/details/`.

---

## 4. What an application receives

| Step of v1 | An application receives |
| --- | --- |
| 1 | **The identifier only**: `sub`, the VSID of the chosen identity for that application. *Built* |
| 2 | The identifier, and the attributes the person accepted to share (§5). *Built* |

An application is **registered** on a server by its administrator: a name shown to people,
the exact addresses it may send people back to, and whether it can keep a secret
([`protocol.md`](protocol.md) §6). *Decided — step 2:* an application can instead
**describe itself** in a document on its own domain, and then signs in the people of any
server that allows it, with no registration anywhere; it declares there what it asks for,
and the administrator can only narrow it. From step 2 its registration also lists what it may ask
for, and which of it the application **requires**; the rest is **optional**. A request for
more than its registration allows is refused. Required or optional is a fact of the
registration, set by the administrator; a request cannot change it.

**An application cannot tell a refusal from an absence.** An attribute the person declined
and one the identity does not have are both simply not in the answer.

---

## 5. The linking page

**Status:** built: the choice of the identity (step 1) and attributes (step 2, core#91,
app#55).

When an application sends a person to brig·id, one page links an identity to it.

- It shows the application's registered name and the account's identities. The identity
  last presented to that application is preselected; with several identities and none
  remembered, nothing is selected for the person.
- It says what the application will receive.
- **From step 2**, when the application asks for attributes: each one is listed **with the
  value that would be sent**, marked required or optional. No box is ticked in advance. An
  optional attribute can be refused and the sign-in continues; refusing a required one stops
  it. What the identity does not have is shown as such, with a field to put it on the
  identity from there; adding a value shares nothing by itself.
- What was decided is remembered per identity and per application, **with the value**: the
  person accepted a value, not a field name. At a later request the attributes accepted come
  already ticked, the ones refused stay unticked, and both say so. One more attribute, or a
  changed value, waits for an answer again: an acceptance of an earlier value counts for
  nothing.
- The page is shown at every sign-in that allows one, as it already was for the choice of
  the identity. A request that asked for no page at all succeeds only when everything asked
  is already settled: accepted with today's value, or optional and refused or absent.
- From the identity's page the person sees, per application, what was shared and when, and
  can withdraw it: one attribute, which is then asked again, or everything, which also
  forgets that this identity was the one presented there. Withdrawing stops future sharing;
  it cannot take back what an application already holds, and the page says so. It does not
  change the identifier either: an application recognises the identity if it comes back.

*Implementation:* `core`, `standing` in `brigid-api/src/routes/authorize.rs` (the one place
that decides what is sent), `brigid-store` (table `grants`); the interface's `/continue/`
and `/identities/details/`. *Verify:* the end-to-end suites of the interface
(`e2e/attributes.spec.ts`) and of the demonstration site.

---

## 6. The life of an account

### Sign-up

**Status:** built.

Sign-up asks for nothing but a passkey: no form, no username to invent. The account exists
only once the passkey is stored, and the session is opened by the same ceremony.

**Who may create an account is the server's sign-up policy** (§7): anyone, the people
invited, or nobody. Where an account cannot be created, sign-in and recovery are untouched,
and the interface says so instead of offering it.

### Onboarding

**Status:** built.

- **The passkey gets a name**, pre-filled with its provider when the authenticator says
  which one it is ("Google Password Manager", "1Password"), editable. The provider is taken
  from an unverified hint and is used for nothing else.
- **Starter identities are proposed, not imposed.** The onboarding proposes a few (for
  example Personal, Work, Public), ticked, that the person can rename, untick or complete.
  The browser creates the ones kept, when the person continues; the server creates none,
  and only says which of its templates are the ones to propose.
- **The recovery code is mandatory.** Onboarding does not finish until the person has
  confirmed keeping it. The server holds its side of it: until an account has a recovery
  code, its identities, their applications and the linking page are closed to it, and
  nothing else is ([`protocol.md`](protocol.md) §7). An account that used its code to come
  back is in the same place until it has taken the new one. A code is replaced, never
  removed. What the server cannot know is whether the person kept the code: that
  confirmation is the interface's.

### Passkeys

**Status:** built.

A person has as many passkeys as they want, names each, and removes the ones retired. The
last one cannot be removed.

- **One passkey per place.** A password manager, a device or a security key keeps one
  passkey for an account, and making a second one there would replace the first. So when
  a passkey is added, the server names the ones the account has, and a place that holds
  one of them refuses. A second passkey is kept somewhere else.
- **Every passkey carries the same name for the account**, the one its first got:
  `swift-otter-4821@id.example.org`. A passkey manager shows one account, of one server.
- **The person says where.** Adding a passkey asks first where it will be kept: this
  device, another phone or tablet, a security key, or wherever the browser proposes. The
  browser is told, so that it does not propose the place that already has one. A place
  that refuses is said in those words, with what to choose instead.

### Recovery

**Status:** built. The change for end-to-end encryption is decided — step 3.

An account whose passkeys are all lost is recovered with its **recovery code**, generated by
the server, never chosen by the person ([`protocol.md`](protocol.md) §7).

- **It does one thing: enrol one new passkey.** No session, no token and no access to
  identities before that passkey exists. Then the code is consumed and a new one is issued.
- It names its account by itself, so nobody is asked for a username they never saw.
- There is **no lockout**: a lockout would let anyone block someone else's recovery, and the
  code has 128 secret bits. Every use is recorded and shown on the account's page.
- **Nobody else can give access back.** No administrator can reset an account (§7): someone
  with neither a passkey nor a recovery code has lost the account. It is deleted and created
  again, with new identities and so new identifiers at every application. Any other answer
  is a way for an administrator into someone's identities.
- *Step 3:* the code also decrypts the account's data
  ([`security-model.md`](security-model.md) §7).

### Leaving

**Status:** built.

- **A person can delete their account**, after a passkey ceremony done for that purpose: the
  session alone is not enough, and a ceremony done to sign in does not count. It removes
  the account, its passkeys, its recovery code, its sessions, its identities and what was
  recorded about the applications they signed in to; no trace of it is kept. The page says
  what it cannot do: an application keeps what it received, and the passkey stays in the
  person's password manager until they remove it there. The last administrator of a server
  names another first (§7).
- **A person can export their data**: their identities with their labels and attributes, and
  the applications each one signed in to, with what was shared, in a documented, readable
  file. **The browser produces it**, from what the account's pages already read: the server
  has no export endpoint, and with end-to-end encryption it could not have one. The format
  is described with the interface (`docs/export-format.md` in `brig-id/app`). It holds no
  secret of the account and none of the identifiers applications know an identity by.

### Sessions

**Status:** built ([`protocol.md`](protocol.md) §5).

Signing in opens a session of the account with this server: one hour unused, twelve hours
in any case. A person can end the session they are in. *Decided — after v1:* seeing and
ending one's other sessions.

---

## 7. Administration

**Status:** built, except the challenge against automated sign-ups (*decided — step 2*).

### Who administers

- **Two roles, on the account**, never on an identity: administrator or not.
- **The first administrator is born from a one-time setup link.** While it has no account,
  the server prints the link at each start, in its console and its log: 128 random bits,
  single use, one hour, stored hashed; a start makes a new one and cancels the one before.
  Opening it creates an account with a passkey, like any sign-up, and that account is the
  administrator. It works whatever the sign-up policy. Reaching the console is the proof of
  being the operator. "The first account created becomes administrator" was rejected:
  whoever reaches a fresh public server first would own it.
- **A server that has accounts and no administrator gets one from the machine.** That is a
  server upgraded from a version without administration, or one whose administrators all
  lost their passkeys. A command (`seedling admin link`) prints a link of the same kind;
  the account that opens it, signed in, receives the role.
- **The secret of a link is in the fragment of its address** (`/setup/#link=…`), for these
  two links and for invitations. A browser never sends a fragment: the secret is in no
  request but the one that uses it, so in no access log of the server or of a proxy in
  front of it.
- **The last administrator stays.** The role is not withdrawn from the last administrator
  who can sign in, and that account is neither suspended nor deleted: another is named
  first. An administrator does not suspend or delete their own account from the
  administration.

### What an administrator can do

- **See the accounts:** the generated username, the creation date, the number of passkeys,
  the last sign-in, whether a recovery code exists, whether the account is suspended or
  administers.
- **Suspend an account.** Its sessions end at once; sign-in, recovery and the tokens its
  applications ask for are refused, and the person is told why. Nothing is deleted: lifted,
  the account is as it was.
- **Delete an account**, with everything a person's own deletion removes (§6).
- **Grant or withdraw the role.**
- **Register applications** (§4): the name shown to people, the return addresses, whether
  the application keeps a secret (shown once, at registration), and what it may ask for.
  Change what an application may ask for, or remove it.
- **Set the sign-up policy** and make invitations (below).
- **Every change requires a passkey ceremony done in the session five minutes ago at most**
  (a server setting). Past it, the server refuses the change and the interface asks for
  the passkey, then sends the change again. Reading asks for nothing more than the session.
- The same can be done from the machine for what a server needs before it has an
  administrator: registering applications and setting the sign-up policy.

### Who may create an account

- **Three policies:** anyone (open), the people invited (by invitation), nobody (closed).
  **A new server is by invitation.** The policy is in the database, not in the
  configuration: an administrator changes it from the interface, the operator from the
  machine (`seedling signup <policy>`).
- **An invitation is a link** that creates a number of accounts chosen by the administrator
  (1 to 100) until a date they choose (1 to 90 days, 7 by default). Like the other links
  it is 128 random bits, stored hashed and shown once. It can be revoked. The server does
  not know who an invitation was given to, and keeps no trace of which account came from
  which invitation.
- An invitation does not open a closed server. The setup link works whatever the policy.
- *Decided — step 2:* for an open server, an **optional challenge against automated
  sign-ups**, self-hosted (a proof-of-work challenge, no third-party captcha), which the
  administrator enables.

### What runs

- **What runs is public.** The server says its edition and its version to anyone, and
  the interface shows them with its own version at the foot of every page:
  `seedling 0.7.1 · interface 0.6.1`. On the machine, the executable says its version.
- That a scanner learns which version to aim at was weighed: someone who runs a server, or
  wonders whether to trust one, should see whether it is current, and a vulnerable
  version is found by trying it anyway.

### What administration never does

- **It never shows** an account's identities, their labels, their attributes, or where they
  are used. No response of the administration holds any of it: what an administrator sees
  of an account is the list above.
- **An administrator cannot give access back to an account** (§6), and cannot act as one.

### The log

- **Every administrative action is recorded**, with those made from the machine: links
  made and used, roles, suspensions, deletions, applications, the sign-up policy,
  invitations.
- **It names accounts by an identifier only.** The usernames are looked up when the log is
  read: an account deleted since is named nowhere, and its entries say only that the
  account is gone. An entry holds what was done, when, by whom, to which account, and a
  short detail (an application's name, the policy chosen); never anything of an identity.
- **It is encrypted** like the rest of the database, and **kept twelve months** by default
  (a server setting): older entries are deleted when the server starts and as it writes
  new ones. A log of who did what to whom is personal data; it is kept as long as it
  serves and no longer.
- The server's operational log (what the process prints) carries no account identifier.

---

## 8. Servers

A passkey works only on the domain that created it, so a person of another server is sent
to that server, which authenticates them.

- **Choosing a server at sign-in.** *Built.* The sign-in page is for the server that serves
  it; another one is named by its domain and the browser goes there.
- **Accounts and identities are never linked across servers.** *Decided; an invariant
  ([`security-model.md`](security-model.md) §8).* An identity lives on one server and works
  only for the applications that server accepts. Someone with a personal server and an
  employer's server has two accounts and nothing connects them: this is what keeps an
  employer from learning about personal identities, and it needs nothing to be built.
- **A single view of one's servers.** *Decided — after v1.* A list kept in the browser only,
  so that no server learns of another.
- **One application, every server.** *Decided — step 2.* An application that describes
  itself ([`protocol.md`](protocol.md) §6) asks the person for their server's domain and
  knows them by that server and their identifier together, so that no server can speak for
  the people of another ([`security-model.md`](security-model.md) §8).
- **"The brig·id network."** *Open.* The idea of servers connected to a network (a directory
  to choose one's server, a federation, or both) has no definition yet; nor has the way a
  domain announces where its server is. Whatever it becomes must hold with the second point.

---

## 9. After v1

Decided as subjects, each needing its own design before anything is built:

- **Services** under "Connected to": a mail relay (an existing provider plugged in by the
  administrator, or a module of the project), an access to an outside service handed to an
  authorised application, SSH certificates signed for an identity. To design first: what the
  server stores, who may obtain what, how the person sees and withdraws it.
- **Facts inherited from a certified account**: an account could carry an official
  certification of the person, from which an identity shares derived facts without the
  details (being of age, an age, being a certified person). A "yes" to "of age" links
  nothing. Where a certification can really come from is to be studied first.
- **A server's own colours** on the sign-in pages, and white-label
  ([`interface.md`](interface.md) §7).
