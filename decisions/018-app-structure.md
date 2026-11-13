# ADR-018 — The three parts of the app: sign-in, account, administration

**Status:** Accepted (2026-11-13, review round 2 with the owner).

## Context

The interface grew page by page: sign-in, signup, recovery, passkeys, identities. It has no
administration at all: a server's accounts can only be managed in its database. Three
audiences use it, with different needs:

- **(a) someone arriving**: sign in, create an account, recover one. Simple, beautiful,
  bare, and in the colours of the server;
- **(b) someone at home**: onboarding, a home page, each identity (appearance, the services
  it is connected to), identities and passkeys;
- **(c) the server's administrator**: a home page, who is an administrator, the accounts.

## Decision

### 1. No names

The three parts are called what they are, everywhere: **sign-in**, **account** and
**administration** (`sign-in`, `account`, `admin` in code and routes). Project names
(Gate, Prism, Helm and others) were considered and not kept: nothing to explain, and no
clash with the Helm of Kubernetes that `forest` will use.

### 2. What belongs where

| | Sign-in | Account | Administration |
| --- | --- | --- | --- |
| Pages | Sign in; choose another server; create an account; recover; the **relying-party step** ("continue to *site* as *identity*", [ADR-005](005-account-sessions-and-rp-tokens.md) §2); the first-administrator setup | Onboarding (passkey name, recovery code); home; identities (create, appearance, connected services, delete); passkeys; recovery code; sessions; settings | Overview; accounts; administrators; registered relying parties; sign-up policy |
| Session | None, or being created | Account session | Account session of an administrator, plus a **recent passkey ceremony** (a few minutes) for any change |
| Routes | `/`, `/signup/`, `/recover/`, `/continue/`, `/setup/` | `/account/…` pages | `/admin/…` pages |
| Themed by | brig·id in v1; the server afterwards ([ADR-016](016-design-system.md)) | brig·id | brig·id |
| Script budget | The smallest: the page works as HTML, a script runs the passkey ceremony | Normal | Normal |

The relying-party step belongs to sign-in: it is what a person sees of brig·id when coming
from another site.

### 3. Architecture: one app, three route groups

- **One Qwik application**, one build, three route groups with their own layouts. Qwik
  already loads code per route and per interaction: three builds would not make sign-in
  lighter, and would triple the release and embedding work
  ([ADR-013](013-leaf-single-executable.md)).
- **Separation is enforced where it counts**, not by bundling: the API authorises every
  request (the administration endpoints require the administrator role and a recent ceremony; the
  interface hiding a page is not a control). Sign-in imports nothing from the other two
  groups, checked by a lint rule, so its pages stay reviewable in one sitting.
- **Themes**: the token levels of ADR-016. One appearance in v1, brig·id's. A server's own
  colours on the sign-in pages, and white-label, come after v1.
- **Static files**: the app stays a static build. What differs per server (name, appearance,
  sign-up policy) is read from one same-origin endpoint and one same-origin stylesheet.

### 4. Roles

Two, on the **account**, never on an identity: administrator or not. Nothing finer until a
real need shows.

What an administrator can do: see the accounts (generated handle, creation date, number of
passkeys, last sign-in, whether a recovery code exists), suspend or delete an account,
invite, grant or withdraw the administrator role, register relying parties, set the sign-up
policy (open, by invitation, closed).

What administration never shows: **an account's identities, their names, or where they are used.** An
operator with the database and the master key can read them
([`security-model.md`](../security-model.md)); the product does not offer it, and an audit
log records administrative actions. The last administrator cannot remove their own role.

**An administrator cannot give access back to an account.** Someone who lost every passkey
and has no recovery code has lost the account: the administrator deletes it and invites the
person again, who starts with new identities, and so new identifiers at every site. Any
other answer is a way for an administrator into someone's identities
([ADR-004](004-account-recovery.md)).

### 5. How the first administrator is born

| Option | Problem |
| --- | --- |
| The first account created becomes administrator | Whoever reaches a fresh public server first owns it. A known class of takeover |
| An administrator named in the configuration | Accounts have no name the operator knows: handles are generated ([ADR-003](003-usernames-and-quick-signup.md)) |
| **A one-time setup link** | — |

**Decision: the one-time setup link.** At its first start with an empty database, the server prints a setup
link in its console and its log (`https://<server>/setup#<token>`: 128 random bits, single
use, valid one hour, stored hashed; a new one is printed at each start while no
administrator exists). Opening it creates an account with a passkey, as any signup, and
that account is the administrator. **Until then, signup is closed.** Reaching the console is
the proof of being the operator.

For the day every administrator's passkeys are lost, the same mechanism from the command
line: `seedling admin invite` (the same command in the three servers) prints a one-time link that grants the role to the account that
redeems it. It needs access to the machine, which is the operator's proof again.

## Consequences

- `core`: an administrator flag on accounts, the setup and invitation tokens, the
  administration API with the recent-ceremony requirement, the audit log, the sign-up
  policy.
- `app`: the three route groups. Sign-in and account are rebuilt on the new design system
  in step 1 of v1; administration is new, in step 2.
- [ADR-004](004-account-recovery.md)'s open point (may an operator reset an account) is
  closed: no.

## References

- [ADR-003](003-usernames-and-quick-signup.md), [ADR-004](004-account-recovery.md),
  [ADR-005](005-account-sessions-and-rp-tokens.md), [ADR-016](016-design-system.md)
