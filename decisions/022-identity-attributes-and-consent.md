# ADR-022 — What a site may receive: identity attributes, shown and accepted on the linking page

**Status:** Proposed (2026-11-13). In scope for milestone 2 of v1
([ADR-014](014-three-products.md)); until then a site receives an identifier and nothing
else ([ADR-008](008-oidc-conformance.md)).

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

## Proposal

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

### 1b. "Connector" covers two things, to be named apart

The word has been used for two different links of an identity:

| | Towards a site that signs people in | Towards a service the identity uses |
| --- | --- | --- |
| What it is | A site the identity signs in to through brig·id | An outside service (a code forge, a task manager, a video platform…) the identity is connected to, so that brig·id can hand an authorised application what it needs to act there, possibly no more than a token |
| brig·id is | The provider: the site trusts it | The client: it holds an access the service granted |
| Exists | With the relying-party flow | Not designed |

Two terms are needed; they are not chosen yet. This ADR uses **site** for the first and
keeps **connector** for the second. **No connector is part of v1.** Each one will need its
own analysis: what the server stores, who may obtain a token, and how the person sees and
withdraws it.

### 2. How a site asks

With the standard OpenID Connect scopes and claims, so that ordinary libraries work:

| Scope | Claims |
| --- | --- |
| `openid` | `sub` |
| `profile` | `given_name`, `family_name`, `preferred_username` |
| `email` | `email`, `email_verified` (§5) |

- A site can only ask for what its registration allows: the administrator who registers it
  lists the scopes it may request. A request for more is refused.
- The claims travel in the ID token. There is no UserInfo endpoint until a real need.

### 3. The linking page

When a site asks for more than the identifier, the page that links the identity to the site
shows:

- the site's registered name, and the identity chosen;
- **each attribute asked for, with the value that would be sent**, one line each;
- what the identity does not have (the site asked, there is nothing to give);
- a way to refuse an attribute and continue, and a way to stop.

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
| B. An address the person types on an identity, sent unverified (`email_verified: false`) | A field | Many applications refuse or distrust an unverified address; the same address on two identities links them |
| C. The same, verified by brig·id (a confirmation message) | The server must send mail: an SMTP account to configure, even on a `seedling` | A dependency and a failure mode for the smallest edition; the server learns a real address |
| D. A relay address generated per identity and per site, forwarded to a real address | The server receives and forwards mail: a domain's mail records, deliverability, abuse handling | A mail service to operate. Out of reach for a `seedling`; conceivable for a hosted offer |
| E. A made-up, undeliverable address derived from the identifier | Nothing | It satisfies a form and breaks every message the application sends. It is a false statement to the site |

**Recommendation:** A for v1 (decided). After v1, **B**, as an attribute like the others,
shown on the linking page with the warning of §4. C only when a server has a reason to send mail
anyway. D as a possible feature of a hosted offer, with its own ADR. E never.

### 6. Order

1. Milestone 1 of v1 ([ADR-014](014-three-products.md)): the identifier only.
2. Milestone 2: attributes (given name, family name, username) and the linking page of §3.
3. After v1: the e-mail question (§5), and connectors (§1b), each with its own record.

## Consequences

- [ADR-002](002-accounts-and-identities.md)'s sentence "an identity is what a relying party
  sees" is to be read as: a site sees an identifier, and what the person accepted to share.
- `core`: attributes on identities, scopes allowed per registered client, the grants
  (identity, site, attributes, values accepted), their withdrawal.
- `app`: the attributes of an identity, the linking page, the per-site view of what was
  shared.
- The public site's wording about what a website learns follows this model.

## References

- OpenID Connect Core 1.0 §5.1 (standard claims), §5.4 (scopes)
- [ADR-002](002-accounts-and-identities.md), [ADR-005](005-account-sessions-and-rp-tokens.md),
  [ADR-008](008-oidc-conformance.md), [ADR-018](018-app-structure.md)
