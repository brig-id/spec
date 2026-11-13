# ADR-025 — What a server publishes about itself: no DID document; post-quantum signatures on request

**Status:** Accepted (2026-11-13, review round 5 with the owner). Amends
[ADR-007](007-identifiers.md) and [ADR-008](008-oidc-conformance.md).

## Context

Two pieces exist without being used.

- **The server's DID document** (`/.well-known/did.json`, crate `brigid-did`). ADR-007 kept
  it when accounts and identities lost their DIDs. It repeats the signing key that the JWKS
  already publishes, and nothing reads it. It is not how an application finds the server:
  that is the OpenID discovery document.
- **The post-quantum primitives** of `brigid-crypto` (X-Wing, composite ML-DSA-65 with
  Ed25519), tested against their drafts' vectors and wired to nothing
  ([`pqc.md`](../pqc.md)). ID tokens are signed with Ed25519.

## Decision

### 1. The DID document is removed

`/.well-known/did.json` and `brigid-did` are removed: less to maintain and to audit. An
application discovers a server through `/.well-known/openid-configuration` and its JWKS,
which stay.

How a domain announces where its brig·id server is, and what a server should publish about
itself for other servers, belongs to the analysis of "the brig·id network"
([ADR-021](021-sharing-between-servers.md)), after v1.

### 2. Composite post-quantum signatures for ID tokens, in v1, on request

- Ed25519 (`EdDSA`) stays the default: every application can verify it.
- A registered application can ask for its ID tokens to be signed with the **composite
  ML-DSA-65 + Ed25519** signature of `brigid-crypto`. The server then holds a second signing
  key, stored and rotated like the first, published in the JWKS and listed in the discovery
  document.
- This amends ADR-008 §2 ("EdDSA only"). There is still no RSA key.
- **Step 3 of v1**, before the feature freeze, so that it is in the audit's scope.

What is known and accepted: the JOSE identifiers for these signatures are still IETF drafts,
so the identifier and the key format follow the draft current at implementation and may
change with it; and almost no application library can verify such a token today. The
feature is ahead of the ecosystem on purpose. The demonstration site verifies one, so that
it is tested end to end.

X-Wing stays unused in v1. Whether it has a role in end-to-end encryption is a question for
that design ([ADR-023](023-end-to-end-encryption.md)), not a commitment.

## Consequences

- `core`: `brigid-did` and its route removed; a composite signing key, the per-application
  algorithm, JWKS and discovery.
- `pqc.md`: phase B (wire signatures to OIDC) is scheduled for step 3 of v1.
- [ADR-007](007-identifiers.md)'s "the server keeps its DID" no longer holds; the rest of it
  does.

## References

- FIPS 204 (ML-DSA); draft-ietf-lamps-pq-composite-sigs
- IETF JOSE working group drafts on ML-DSA and composite signatures (identifiers to take
  from the current draft at implementation)
- [`pqc.md`](../pqc.md), [ADR-007](007-identifiers.md), [ADR-008](008-oidc-conformance.md)
