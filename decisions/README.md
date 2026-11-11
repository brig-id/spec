# Architecture Decision Records — brig·id

This directory contains Architecture Decision Records (ADRs) for brig·id.

An ADR documents a significant technical decision: the context, the options considered,
the decision made, and the consequences.

## Index

| ADR | Title | Status |
|---|---|---|
| [001](001-ring-eddsa-decoding-key-format.md) | `jsonwebtoken` + ring: EdDSA DecodingKey expects raw 32-byte public key | Accepted |
| [002](002-accounts-and-identities.md) | Accounts and identities | Accepted |
| [003](003-usernames-and-quick-signup.md) | Usernames and passkey-only quick signup | Proposed |
| [004](004-account-recovery.md) | Account recovery when every passkey is lost | Proposed |

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
