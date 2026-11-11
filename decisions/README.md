# Architecture Decision Records — brig·id

This directory contains Architecture Decision Records (ADRs) for brig·id.

An ADR documents a significant technical decision: the context, the options considered,
the decision made, and the consequences.

## Index

| ADR | Title | Status |
|---|---|---|
| [001](001-ring-eddsa-decoding-key-format.md) | `jsonwebtoken` + ring: EdDSA DecodingKey expects raw 32-byte public key | Accepted |
| [002](002-accounts-and-identities.md) | Accounts and identities | Accepted |
| [003](003-usernames-and-quick-signup.md) | Usernames and passkey-only quick signup | Accepted |
| [004](004-account-recovery.md) | Account recovery when every passkey is lost | Accepted |
| [005](005-account-sessions-and-rp-tokens.md) | Account sessions and relying-party tokens are separate things | Accepted |
| [006](006-vsid-secret.md) | The VSID secret is stored, not derived from the master key | Accepted |
| [007](007-identifiers.md) | Identifiers: UUIDs for accounts and identities, a DID for the server only | Accepted |

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
