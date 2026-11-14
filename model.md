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

**Status:** decided — step 2. Stored from the start so that end-to-end encryption
([`security-model.md`](security-model.md) §7) takes them over at step 3 without a second
migration.

Attributes are values a person puts on an identity in order to share them. They are
optional and typed, and they belong to one identity: nothing is copied from one identity to
another, and the account has none.

| Attribute | OpenID Connect claim | Scope |
| --- | --- | --- |
| First name, last name | `given_name`, `family_name` | `profile` |
| A username chosen for this identity | `preferred_username` | `profile` |
| A photo or avatar | `picture` | `profile` |
| Language, time zone | `locale`, `zoneinfo` | `profile` |
| An e-mail address | `email`, `email_verified` | `email` |

- The username attribute is a handle shown to applications, not a name to sign in with.
- **E-mail** is optional, and the interface recommends it, because many applications need
  one. If the server's administrator configured outgoing mail, the address is confirmed by a
  message and shared as verified; otherwise it is shared as unverified. A small server is not
  required to send mail. A made-up, undeliverable address is never generated: it would be a
  false statement to the application.

**Attributes can link identities.** The per-application identifier keeps two identities
apart; the same last name, username or address given through two identities undoes that,
and no protocol can prevent it. The interface says so where an attribute is filled in, and
warns clearly when an address is already used on another identity of the account (a variant
of the same mailbox, `name+work@…`, protects little). It lets the person decide.

---

## 4. What an application receives

| Step of v1 | An application receives |
| --- | --- |
| 1 | **The identifier only**: `sub`, the VSID of the chosen identity for that application. *Built* |
| 2 | The identifier, and the attributes the person accepted to share (§5). *Decided* |

An application is **registered** on a server by its administrator: a name shown to people,
the exact addresses it may send people back to, and whether it can keep a secret
([`protocol.md`](protocol.md) §6). From step 2 its registration also lists what it may ask
for, and which of it the application **requires**; the rest is **optional**. A request for
more than its registration allows is refused.

---

## 5. The linking page

**Status:** step 1, the choice of the identity, *built*. Step 2, attributes, *decided*.

When an application sends a person to brig·id, one page links an identity to it.

- It shows the application's registered name and the account's identities. The identity
  last presented to that application is preselected; with several identities and none
  remembered, nothing is selected for the person.
- It says what the application will receive.
- **From step 2**, when the application asks for attributes: each one is listed **with the
  value that would be sent**, marked required or optional. No box is ticked in advance. An
  optional attribute can be refused and the sign-in continues; refusing a required one stops
  it. What the identity does not have is shown as such.
- What was accepted is remembered per identity and per application. A later request for the
  same attributes passes without the page; one more attribute, or a changed value, shows it
  again: the person accepted a value, not a field name.
- A request that asked for no page at all succeeds only when everything asked was already
  accepted.
- From the identity's page the person sees, per application, what was shared and when, and
  can withdraw it. Withdrawing stops future sharing; it cannot take back what an
  application already holds, and the page says so.

---

## 6. The life of an account

### Sign-up

**Status:** built.

Sign-up asks for nothing but a passkey: no form, no username to invent. The account exists
only once the passkey is stored, and the session is opened by the same ceremony.

A server setting **closes sign-up** (`security.signup_open`): the creation of accounts is
refused, sign-in and recovery are untouched, and the interface says so instead of offering
it. The project's own server needs it before it has any administration. *Decided — step 2:* the administrator's sign-up policy
(§7).

### Onboarding

**Status:** the passkey's name and the recovery code are built; the rest is decided — step 2.

- **The passkey gets a name**, pre-filled with its provider when the authenticator says
  which one it is ("Google Password Manager", "1Password"), editable. The provider is taken
  from an unverified hint and is used for nothing else.
- **Starter identities are proposed, not imposed.** The onboarding proposes a few (for
  example Personal, Work, Public), ticked, that the person can rename, untick or complete.
  The browser creates the ones kept; the server creates none. *Built today:* the server
  creates three.
- **The recovery code is mandatory.** Onboarding does not finish until the person has
  confirmed keeping it. *Built today:* it is offered and optional.

### Passkeys

**Status:** built.

A person adds a passkey per device, names each, and removes the ones retired. The last one
cannot be removed.

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

**Status:** decided — step 2.

- **A person can delete their account**, after a passkey ceremony done for that purpose. It
  removes the account, its passkeys, sessions, identities and what was recorded about the
  applications they signed in to. The page says what it cannot do: an application keeps
  what it received. The last administrator of a server names another first.
- **A person can export their data**: their identities with their labels and attributes, and
  the applications each one signed in to, in a documented, readable file. With end-to-end
  encryption, the browser produces it.

### Sessions

**Status:** built ([`protocol.md`](protocol.md) §5).

Signing in opens a session of the account with this server: one hour unused, twelve hours
in any case. A person can end the session they are in. *Decided — after v1:* seeing and
ending one's other sessions.

---

## 7. Administration

**Status:** decided — step 2. Today a server is administered from its command line and its
database.

- **Two roles, on the account**, never on an identity: administrator or not.
- **The first administrator is born from a one-time setup link.** At its first start with an
  empty database the server prints the link in its console and its log (128 random bits,
  single use, one hour, stored hashed, renewed at each start while no administrator exists).
  Opening it creates an account with a passkey, like any sign-up, and that account is the
  administrator. Until then sign-up is closed. Reaching the console is the proof of being the
  operator. "The first account created becomes administrator" was rejected: whoever reaches
  a fresh public server first would own it.
- If every administrator's passkeys are lost, a command on the machine prints a one-time
  link that grants the role to the account that redeems it.
- **What an administrator can do:** see the accounts (generated username, creation date,
  number of passkeys, last sign-in, whether a recovery code exists), suspend or delete an
  account, invite, grant or withdraw the role, register applications, set the sign-up policy
  (open, by invitation, closed). Every change requires a passkey ceremony a few minutes old
  at most.
- **What administration never shows:** an account's identities, their labels, their
  attributes, or where they are used. An audit log records administrative actions.
- **An administrator cannot give access back to an account** (§6).
- For an open server: an **optional challenge against automated sign-ups**, self-hosted (a
  proof-of-work challenge, no third-party captcha), which the administrator enables.

---

## 8. Servers

A passkey works only on the domain that created it, so a person of another server is sent
to that server, which authenticates them.

- **Choosing a server at sign-in.** *Built.* The sign-in page is for the server that serves
  it; another one is named by its domain and the browser goes there.
- **Accounts and identities are never linked across servers.** *Decided; an invariant
  ([`security-model.md`](security-model.md) §8).* An identity lives on one server and works
  only for the applications registered on that server. Someone with a personal server and an
  employer's server has two accounts and nothing connects them: this is what keeps an
  employer from learning about personal identities, and it needs nothing to be built.
- **A single view of one's servers.** *Decided — after v1.* A list kept in the browser only,
  so that no server learns of another.
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
