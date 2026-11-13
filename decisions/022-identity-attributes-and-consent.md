# ADR-022 — What an application may receive: identity attributes, shown and accepted on the linking page

**Status:** Accepted (2026-11-13, review round 4 with the owner). Built at step 2 of v1
([ADR-014](014-three-products.md)); until then an application receives an identifier and
nothing else ([ADR-008](008-oidc-conformance.md)).

## Context

[ADR-002](002-accounts-and-identities.md) describes an identity by a name, a logo and a
description, and calls it "what a relying party sees". The review of 2026-11-13 made the
model more precise:

- **The name and the icon of an identity are labels inside brig·id.** They help the person
  tell their identities apart. No site ever needs them, and no site receives them.
- **An identity can also carry information about the person**: a first name, a last name, a
  username, connectors. A site may ask for some of it.
- **Nothing is shared without the person seeing exactly what, and accepting it**, on the
  page that links the identity to the site.

Separately: an identity has no e-mail address, by design, and many applications want one to
create an account. An analysis was asked before anything is built.

## Decision

### 1. Two kinds of data on an identity

| | Label | Attributes |
| --- | --- | --- |
| What | Name, icon, description | Values the person enters in order to share them |
| Seen by | The person, in brig·id | A site, only after consent (§3) |
| Required | Name and icon | None: an identity can have no attribute at all |
| Examples | "Work", a briefcase | Given name, family name, username |

Attributes are optional, typed, and stored encrypted like the other fields of an identity.
They belong to one identity: nothing is copied from one identity to another, and the
account has none.

### 1b. Two words in place of "connector"

"Connector" had been used for two different links of an identity. They are named apart, in
the interface and in the documents:

| | **Application** | **Service** |
| --- | --- | --- |
| What it is | What an identity signs in to through brig·id: a website, a mobile or desktop application | An outside service the identity is connected to (a code forge, a task manager, a mail relay…), so that brig·id can hand an authorised application what it needs there, possibly no more than a token |
| brig·id is | The provider: the application trusts it | The client: it holds an access the service granted, or operates the service |
| Exists | With the relying-party flow | Not designed |

Other words may join these two later. **No service is part of v1.** Each one will need its
own record: what the server stores, who may obtain what, and how the person sees and
withdraws it.

### 2. The attributes of v1, and how an application asks

With the standard OpenID Connect scopes and claims, so that ordinary libraries work:

| Scope | Claims | Attribute |
| --- | --- | --- |
| `openid` | `sub` | (the identifier) |
| `profile` | `given_name`, `family_name` | First name, last name |
| `profile` | `preferred_username` | A username chosen for this identity |
| `profile` | `picture` | A photo or avatar chosen by the person |
| `profile` | `locale`, `zoneinfo` | Language, time zone |
| `email` | `email`, `email_verified` | An address (§5) |

- An application can only ask for what its registration allows: the administrator who
  registers it lists what it may request, and **which of it the application cannot do
  without**. A request for more is refused.
- The claims travel in the ID token. There is no UserInfo endpoint until a real need.

### 3. The linking page

When a site asks for more than the identifier, the page that links the identity to the site
shows:

- the site's registered name, and the identity chosen;
- **each attribute asked for, with the value that would be sent**, one line each;
- what the identity does not have (the site asked, there is nothing to give);
- for each attribute, whether the application declared it **required** or **optional**: an
  optional one can be refused and the sign-in continues; refusing a required one stops it;
- a way to stop.

Rules:

- No box is ticked in advance on the person's behalf, and nothing is sent by default beyond
  the identifier.
- What was accepted is remembered per identity and per site. A later request for the same
  attributes passes without the page; a request for one more shows the page again.
- If an accepted value changes, the next sign-in shows the page again: the person accepted a
  value, not a field name.
- A silent request (`prompt=none`) succeeds only when everything asked was already accepted.
- From the identity's page the person sees, per site, what was shared and when, and can
  withdraw it. Withdrawing stops future sharing; it cannot take back what a site already
  holds, and the page says so.

### 4. Attributes link identities

The per-site identifier keeps two identities apart. An attribute can undo that: the same
family name, username or e-mail address given through two identities lets two sites, or one
site, connect them. This cannot be solved by the protocol. The interface says it where the
person fills an attribute in, and warns when a value is already used on another identity of
the account.

### 5. E-mail

| Option | What it takes | Problem |
| --- | --- | --- |
| A. None | Nothing | Applications that require an address do not work, or ask for it themselves |
| B. An address the person types on an identity | A field; a way to send mail if it is to be verified | The same address on two identities links them |
| C. A relay address per identity, forwarded to a real one | Receiving and forwarding mail: a domain's mail records, deliverability, abuse handling | A mail service to operate |
| D. A made-up, undeliverable address | Nothing | A false statement to the application, and every message it sends is lost |

**Decided:**

- **B, at step 2.** An identity can carry an address. It is optional, and the interface
  recommends it, because so many applications need one.
- **Verified when the server can send mail.** If the administrator configured an outgoing
  mail account, the address is confirmed by a message and shared as verified; otherwise it
  is shared as unverified. A small server is not required to send mail.
- **The same address on two identities:** the interface warns clearly that it links them,
  says that a variant of the same mailbox (`name+work@…`) protects little, and lets the
  person decide.
- **C is a service, later**: either an existing relay provider that the administrator plugs
  in, or a module of the project. The administrator enables it or not. It removes the
  linking problem of B, and gets its own record.
- D never.

### 5b. Facts inherited from a certified account

An account could one day carry an official certification of the person, from which an
identity would share **derived facts** without the details: being of age, an age, being a
certified person. A yes to "of age" links nothing, which fits the model well.

**After v1, with its own record, analysis first**: where a certification can really come
from (a state digital identity, the European identity wallet, a verification provider, a
server's administrator) is to be studied before anything is chosen.

### 6. Order

1. Step 1 of v1: the identifier only.
2. Step 2: the attributes of §2, required and optional, the linking page of §3, the
   address of §5. They are stored so that end-to-end encryption
   ([ADR-023](023-end-to-end-encryption.md)) can take them over at step 3 without a second
   migration.
3. After v1: services (§1b), among them the mail relay; facts inherited from a certified
   account (§5b).

## Consequences

- [ADR-002](002-accounts-and-identities.md)'s sentence "an identity is what a relying party
  sees" is to be read as: an application sees an identifier, and what the person accepted
  to share.
- `core`: attributes on identities, scopes allowed per registered client, the grants
  (identity, site, attributes, values accepted), their withdrawal.
- `app`: the attributes of an identity, the linking page, the per-site view of what was
  shared.
- The public site's wording about what a website learns follows this model.

## References

- OpenID Connect Core 1.0 §5.1 (standard claims), §5.4 (scopes)
- [ADR-002](002-accounts-and-identities.md), [ADR-005](005-account-sessions-and-rp-tokens.md),
  [ADR-008](008-oidc-conformance.md), [ADR-018](018-app-structure.md)
