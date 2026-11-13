# ADR-018 — The three parts of the app: Gate, Prism, Helm

**Status:** Proposed (2026-11-13). The names and the birth of the first administrator are the
points to decide.

## Context

The interface grew page by page: sign-in, signup, recovery, passkeys, identities. It has no
administration at all: a server's accounts can only be managed in its database. Three
audiences use it, with different needs:

- **(a) someone arriving**: sign in, create an account, recover one. Simple, beautiful,
  bare, and in the colours of the server;
- **(b) someone at home**: onboarding, a home page, each identity (appearance, the services
  it is connected to), identities and passkeys;
- **(c) the server's administrator**: a home page, who is an administrator, the accounts.

## Proposal

### 1. Names

The names are for the project (code, routes, documentation, design reviews). **The interface
itself keeps plain words**: "Sign in", "Your account", "Administration".

| Part | Proposed | Why | Alternatives |
| --- | --- | --- | --- |
| (a) arriving | **Gate** | The way in. Short, and says what must stay small | Threshold, Gangway |
| (b) at home | **Prism** | Where one private self becomes several identities: the figure of the public site | Home, Hearth (Brigid is the goddess of the hearth), Cabin |
| (c) administration | **Helm** | Who steers the ship; it is already the name of the reference server (`helm.brig.id`) | Bridge, Wheelhouse, Canopy |

One caveat: **Helm** is also the name of Kubernetes' package manager, which `forest` will
use. "Install the Helm chart, then open the Helm" is a sentence to avoid. If that weighs,
**Bridge** is the second choice for (c), though "bridge" also suggests a connector.

A consistent maritime set (Gangway, Cabin, Helm) or a plain one (Gate, Home, Admin) are both
defensible; the recommendation mixes them for the sake of the two strongest names.

### 2. What belongs where

| | Gate | Prism | Helm |
| --- | --- | --- | --- |
| Pages | Sign in; choose another server; create an account; recover; the **relying-party step** ("continue to *site* as *identity*", [ADR-005](005-account-sessions-and-rp-tokens.md) §2); the first-administrator setup | Onboarding (passkey name, recovery code); home; identities (create, appearance, connected services, delete); passkeys; recovery code; sessions; settings | Overview; accounts; administrators; registered relying parties; sign-up policy; appearance of the Gate |
| Session | None, or being created | Account session | Account session of an administrator, plus a **recent passkey ceremony** (a few minutes) for any change |
| Routes | `/`, `/signup`, `/recover`, `/authorize`, `/setup` | `/home/…` | `/admin/…` |
| Themed by | The server ([ADR-016](016-design-system.md)) | brig·id (white-label on `forest` only) | brig·id |
| Script budget | The smallest: the page works as HTML, a script runs the passkey ceremony | Normal | Normal |

The relying-party step belongs to the Gate: it is what a person sees of brig·id when coming
from another site, and it is the page each server will want in its colours.

### 3. Architecture: one app, three route groups

- **One Qwik application**, one build, three route groups with their own layouts. Qwik
  already loads code per route and per interaction: three builds would not make the Gate
  lighter, and would triple the release and embedding work
  ([ADR-013](013-leaf-single-executable.md)).
- **Separation is enforced where it counts**, not by bundling: the API authorises every
  request (the administration endpoints require the administrator role and a recent ceremony; the
  interface hiding a page is not a control). The Gate imports nothing from the other two
  groups, checked by a lint rule, so its pages stay reviewable in one sitting.
- **Themes**: the token levels of ADR-016. The server's appearance (accent, logo, name,
  background) applies to the Gate; a `forest` white-label theme applies to the three.
- **Static files**: the app stays a static build. What differs per server (name, appearance,
  sign-up policy) is read from one same-origin endpoint and one same-origin stylesheet.

### 4. Roles

Two, on the **account**, never on an identity: administrator or not. Nothing finer until a
real need shows.

What an administrator can do: see the accounts (generated handle, creation date, number of
passkeys, last sign-in, whether a recovery code exists), suspend or delete an account,
invite, grant or withdraw the administrator role, register relying parties, set the sign-up
policy (open, by invitation, closed) and the Gate's appearance.

What Helm never shows: **an account's identities, their names, or where they are used.** An
operator with the database and the master key can read them
([`security-model.md`](../security-model.md)); the product does not offer it, and an audit
log records administrative actions. The last administrator cannot remove their own role.

To decide before building: whether an administrator can reset an account that lost both its
passkeys and its recovery code ([ADR-004](004-account-recovery.md) leaves it open). The
recommendation is no reset: delete and re-invite, since a reset is a way into someone's
identities.

### 5. How the first administrator is born

| Option | Problem |
| --- | --- |
| The first account created becomes administrator | Whoever reaches a fresh public server first owns it. A known class of takeover |
| An administrator named in the configuration | Accounts have no name the operator knows: handles are generated ([ADR-003](003-usernames-and-quick-signup.md)) |
| **A one-time setup link** | — |

**Recommendation.** At its first start with an empty database, the server prints a setup
link in its console and its log (`https://<server>/setup#<token>`: 128 random bits, single
use, valid one hour, stored hashed; a new one is printed at each start while no
administrator exists). Opening it creates an account with a passkey, as any signup, and
that account is the administrator. **Until then, signup is closed.** Reaching the console is
the proof of being the operator.

For the day every administrator's passkeys are lost, the same mechanism from the command
line: `leaf admin invite` prints a one-time link that grants the role to the account that
redeems it. It needs access to the machine, which is the operator's proof again.

## Consequences

- `core`: an administrator flag on accounts, the setup and invitation tokens, the
  administration API with the recent-ceremony requirement, the audit log, the sign-up policy.
- `app`: the three route groups; Helm is new; the Gate is redrawn on the new design system.
- For v1: Gate and Prism complete, Helm minimal (accounts, administrators, relying parties,
  sign-up policy). Appearance settings come after v1.

## References

- [ADR-003](003-usernames-and-quick-signup.md), [ADR-004](004-account-recovery.md),
  [ADR-005](005-account-sessions-and-rp-tokens.md), [ADR-016](016-design-system.md)
