# Architecture Decision Records — brig·id

This directory contains Architecture Decision Records (ADRs) for brig·id.

An ADR documents a significant technical decision: the context, the options considered,
the decision made, and the consequences.

## Index

| ADR | Title | Status |
|---|---|---|
| [001](001-ring-eddsa-decoding-key-format.md) | `jsonwebtoken` + ring: EdDSA DecodingKey expects raw 32-byte public key | Deprecated |
| [002](002-accounts-and-identities.md) | Accounts and identities | Accepted |
| [003](003-usernames-and-quick-signup.md) | Usernames and passkey-only quick signup | Accepted |
| [004](004-account-recovery.md) | Account recovery when every passkey is lost | Accepted |
| [005](005-account-sessions-and-rp-tokens.md) | Account sessions and relying-party tokens are separate things | Accepted |
| [006](006-vsid-secret.md) | The VSID secret is stored, not derived from the master key | Accepted |
| [007](007-identifiers.md) | Identifiers: UUIDs for accounts and identities, a DID for the server only | Accepted |
| [008](008-oidc-conformance.md) | The relying-party flow for v1; OpenID conformance testing and certification are not now | Accepted |
| [009](009-passkeys-only-user-verification.md) | Authentication: passkeys only, user verification required, no attestation | Accepted |
| [010](010-third-party-audit-funding.md) | Third-party security audit: NLnet first, OSTIF second, public call later | Accepted |
| [011](011-releases-and-dependency-flow.md) | Releases: tagged versions everywhere, release-please, Renovate | Accepted |
| [012](012-self-contained-app.md) | The app is self-contained: nothing external at run time or at build time | Accepted |
| [013](013-leaf-single-executable.md) | The smallest edition is a single executable | Accepted |
| [014](014-three-products.md) | Three products: seedling, grove, forest | Accepted |
| [015](015-icons.md) | Icons: Phosphor, embedded, for the interface and for identity logos | Accepted |
| [016](016-design-system.md) | Design system: our own tokens and components on native HTML, on Qwik 2 | Accepted |
| [017](017-typography.md) | Typography: Geist and Geist Mono, embedded | Accepted |
| [018](018-app-structure.md) | The three parts of the app: sign-in, account, administration | Accepted |
| [019](019-editions.md) | Editions: what seedling, grove and forest are made of | Accepted |
| [020](020-licensing-and-pricing.md) | Licences and pricing: AGPL alone, a registered name, paid services | Accepted |
| [021](021-sharing-between-servers.md) | Identities across servers: no linked accounts, nothing shared between servers | Accepted |
| [022](022-identity-attributes-and-consent.md) | What an application may receive: identity attributes, shown and accepted on the linking page | Accepted |
| [023](023-end-to-end-encryption.md) | What describes a person is encrypted end to end, with a key only their passkey gives | Accepted (objective of v1; design completed at step 2) |
| [024](024-account-lifecycle.md) | The life of an account: starting, recovering, leaving | Accepted |
| [025](025-server-surface.md) | What a server publishes about itself: no DID document; post-quantum signatures on request | Accepted |

A **Proposed** ADR is an analysis with a recommendation. It binds nothing until its status
changes to Accepted.

## Format

Each ADR follows this structure:

```
# ADR-NNN — Title

**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-NNN

## Context
## Decision
## Consequences
## References
```
