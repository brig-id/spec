# ADR-014 — Three products: leaf, grove, forest

**Status:** Accepted (2026-11-13). What each product contains is analysed in
[ADR-019](019-editions.md) (Proposed).

## Context

Three server names exist: `leaf` (the only one with code), `grove` and `forest` (empty
repositories). They were described as sizes of one thing. The direction is now that they are
three products, for three situations: a personal server, an organisation's server, a
company's installation.

## Decision

### 1. Three deliverables

`leaf`, `grove` and `forest` are three products, each with its repository, its releases, its
documentation and its tests. They are not three settings of one binary: an operator chooses
a product, and going from one to another is a migration (export, import), not a
configuration flag.

### 2. One model

What an account, an identity, a passkey, a session and a token are does not depend on the
product. Protocol, cryptography and the security model live in `crypto`, `core` and `app`,
which are not products. No edition weakens the model, and a security fix reaches the three.

To keep three products within what a small project can maintain, a product repository
contains only what differs: packaging, storage backend, listeners, clustering,
operations. A feature that needs different *protocol* behaviour belongs to `core`.

### 3. Where today's code goes

Today's `leaf` (a container image behind a proxy, a configuration file, metrics, probes) is
closer to what `grove` will be than to the new `leaf`.

- **`leaf`** becomes the single executable of [ADR-013](013-leaf-single-executable.md):
  simpler to run than today.
- **`grove`** starts from today's `leaf` deployment: the image, the compose file, the proxy.
- **`forest`** is designed when a first installation needs it.

### 4. Order

1. **v1 is `leaf`**: the single executable, with the relying-party flow
   ([ADR-005](005-account-sessions-and-rp-tokens.md) §2,
   [ADR-008](008-oidc-conformance.md)) and the self-contained interface
   ([ADR-012](012-self-contained-app.md)).
2. `grove` after v1.
3. `forest` after `grove`, on demand.

The reference server `helm.brig.id` runs `leaf`'s image until `grove` exists, then becomes a
`grove`: a public server for many people is what `grove` is for.

## Consequences

- `server-grove` and `server-forest` stay empty until their turn; their README files
  describe intent, not features.
- The public site keeps saying what exists: `leaf` only.
- An export/import format between products is needed before `grove` ships
  ([ADR-019](019-editions.md)).

## References

- [ADR-013](013-leaf-single-executable.md), [ADR-019](019-editions.md),
  [ADR-020](020-licensing-and-pricing.md)
