# Protocol — brig·id

> **Audience:** external security auditors and advanced contributors.
> This document describes the WebAuthn registration/authentication flows,
> OIDC ID Token issuance, VSID computation, and cryptographic parameters.

---

## 1. Cryptographic Parameters

| Purpose | Algorithm | Key size | Standard |
|---|---|---|---|
| Symmetric encryption | AES-256-GCM | 256-bit key, 96-bit nonce | NIST SP 800-38D |
| Key derivation | HKDF | 256-bit output | RFC 5869 |
| Hash function (HKDF + general) | SHA3-256 | 256-bit digest | FIPS 202 |
| Classical signatures (OIDC) | Ed25519 | 256-bit private key | RFC 8032 |
| PQC KEM | ML-KEM-768 + X25519 (hybrid) | 1184 + 32 = 1216-byte combined public key | FIPS 203 |
| PQC signatures | ML-DSA-65 + Ed25519 (hybrid) | 1952 + 32 = 1984-byte combined public key | FIPS 204 |

---

## 2. VSID Computation

VSID (Verifiable Subject Identifier) is the stable, opaque `sub` claim in OIDC tokens.
It ensures that the same user has a **different identifier per relying party**,
preventing cross-RP identity correlation.

### Formula

```
VSID = HKDF-SHA3-256(
    ikm  = did_root_bytes,     // UTF-8 bytes of the user's root DID
    salt = random_salt,         // 32 random bytes, stored encrypted per user
    info = "brigid-vsid:" || client_id
)
→ encoded as lowercase hex string (64 chars)
```

### Invariants

1. Same `(did_root, client_id, salt)` → same VSID (deterministic).
2. Different `client_id` → different VSID (unlinkable across RPs).
3. VSID must **never** be derived from an alias or a virtual identity.
4. VSID must **never** be derived from a username directly.

*Reference implementation:* [`brigid-identity/src/vsid.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-identity/src/vsid.rs) — `compute_vsid()`

---

## 3. WebAuthn Registration Flow

```
Client                          brigid-api                      brigid-store
  │                                 │                               │
  │─── POST /auth/register/begin ──▶│                               │
  │    { username: "user@host" }     │                               │
  │                                 │ validate username format      │
  │                                 │ generate challenge (random)   │
  │                                 │ store challenge (encrypted)──▶│
  │◀── 200 { challenge, rp, ... } ──│                               │
  │                                 │                               │
  │  [user creates passkey on device]│                              │
  │                                 │                               │
  │─── POST /auth/register/finish ─▶│                               │
  │    { credential response }      │                               │
  │                                 │ retrieve challenge ◀──────────│
  │                                 │ verify attestation            │
  │                                 │ verify RP ID == server domain │
  │                                 │ verify signature              │
  │                                 │ store credential (encrypted)─▶│
  │◀── 200 OK ──────────────────────│                               │
```

### Security checks (registration)
- RP ID must exactly match the server domain (no wildcard).
- Challenge must be single-use (deleted after verification).
- Attestation verified if attestation conveyance is `direct` or `enterprise`.

---

## 4. WebAuthn Authentication Flow

```
Client                          brigid-api                      brigid-store
  │                                 │                               │
  │─── POST /auth/login/begin ─────▶│                               │
  │    { username: "user@host" }    │                               │
  │                                 │ look up credentials ◀─────────│
  │                                 │ generate challenge            │
  │                                 │ store challenge (encrypted)──▶│
  │◀── 200 { challenge, allowCredentials }                          │
  │                                 │                               │
  │  [user approves on device]      │                               │
  │                                 │                               │
  │─── POST /auth/login/finish ────▶│                               │
  │    { assertion response }       │                               │
  │                                 │ retrieve challenge + cred ◀───│
  │                                 │ verify assertion signature    │
  │                                 │ verify signature counter ↑    │
  │                                 │ update counter (encrypted)───▶│
  │                                 │ compute VSID                  │
  │                                 │ issue OIDC ID Token           │
  │◀── 200 { id_token } ───────────│                               │
```

### Security checks (authentication)
- Challenge must be single-use (deleted after verification).
- Signature counter must be strictly greater than stored value (counter replay detection).
- RP ID verified against server domain.

---

## 5. OIDC ID Token

### Token structure (JWT)

```json
{
  "alg": "EdDSA",
  "typ": "JWT",
  "kid": "<uuid>"
}
```

```json
{
  "sub":        "<vsid>",           // VSID — never username, alias, or DID
  "iss":        "https://example.com",
  "aud":        "<client_id>",
  "exp":        1234567890,
  "iat":        1234567890,
  "jti":        "<uuid>",            // unique per token — replay prevention
  "server":     "example.com",       // issuing server hostname — not user-identifying
  "alias_type": "root"
}
```

The ID token deliberately omits the user's root DID (or any other stable,
cross-RP user identifier). `sub` is a VSID derived from
`(did_root, client_id, salt)`, which makes it pairwise per relying party.
Including a stable DID in any claim would let two colluding RPs correlate
the same user across `aud` values and undermine the pairwise-subject
privacy guarantee. `iss` and `server` identify the **issuer**, not the
subject, and are therefore safe to publish.

### Token validation (brigid-oidc)

`brigid-oidc` exposes **two** validation paths with different replay semantics;
choosing the right one is part of the protocol contract.

#### Path A — `validate_token` (single-use consumption)

Used when a token is meant to be consumed exactly once (e.g. a one-shot bearer
hand-off, or any future flow where the relying party should not replay a
previously seen token):

1. Verify EdDSA signature using the server's `OidcSigningKey`.
2. Check `exp` > now (no clock leeway).
3. Check `iss` == expected issuer.
4. Check `aud` == expected `client_id`.
5. Atomically check `jti` not in `JtiStore` **and** insert it with TTL = `exp`.
   A subsequent call with the same `jti` is rejected as a replay.

#### Path B — `decode_token` (bearer-style, reusable until `exp` or logout)

Used by the `AuthenticatedClaims` extractor in `brigid-api` middleware to gate
authenticated routes (e.g. `POST /auth/logout`). A bearer token must remain
valid for multiple successive requests until it expires or is revoked, so this
path does **not** insert into `JtiStore`:

1. Verify EdDSA signature.
2. Check `exp`, `iss`, `aud` as in Path A (no clock leeway).
3. Check `jti` is **not** in `JtiStore::is_blacklisted()` (revocation check).
   Logout writes the `jti` into the blacklist with TTL = `exp`, so any further
   bearer use after logout is rejected.

ID Tokens issued at `/auth/login/finish` are returned to the client as bearer
credentials and are validated on every subsequent request via Path B. There is
currently **no** production endpoint that calls Path A; `validate_token` is
provided for future flows that need single-use semantics. The shared `JtiStore`
remains bounded because both paths key on `jti` with `TTL = exp` and expired
entries are evicted on access.

*Reference:* [`brigid-oidc/src/token.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-oidc/src/token.rs) — `validate_token()`, `decode_token()`;
[`brigid-api/src/middleware.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-api/src/middleware.rs) — `AuthenticatedClaims` extractor (Path B).

---

## 6. Discovery Endpoints

### `GET /.well-known/openid-configuration`

Limited OIDC Discovery document (non-standard profile). brig·id uses WebAuthn
instead of the standard OAuth 2.0 Authorization Code flow. Standard OIDC fields
that do not apply to the WebAuthn flow are omitted from the response.

| Field | Value | Notes |
|---|---|---|
| `issuer` | `https://{domain}` | |
| `jwks_uri` | `https://{domain}/.well-known/jwks.json` | |
| `response_types_supported` | `["id_token"]` | |
| `subject_types_supported` | `["pairwise"]` | VSID is pairwise — different per relying party |
| `id_token_signing_alg_values_supported` | `["EdDSA"]` | |
| `authorization_endpoint` | *omitted* | Not applicable — use `POST /auth/login/begin` |
| `token_endpoint` | *omitted* | Not applicable — use `POST /auth/login/finish` |

### `GET /.well-known/jwks.json`

JWK Set containing the Ed25519 public key used for token signing.

```json
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "x":   "<base64url of 32-byte raw public key>",
    "kid": "<uuid>",
    "alg": "EdDSA",
    "use": "sig"
  }]
}
```

**Important implementation note:** `jsonwebtoken v9` (ring backend) expects the
`DecodingKey` to be initialised with raw 32-byte public key bytes, **not**
SubjectPublicKeyInfo DER. See [`brigid-oidc/src/key.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-oidc/src/key.rs) — `decoding_key()`.

---

## 7. DID Web Document

### `GET /.well-known/did.json`

Exposes the server's DID Web document, enabling DID-based identity resolution.

*Reference:* [`brigid-did/src/web.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-did/src/web.rs) — `did_web_document()`

---

## 8. Logout Flow

```
Client                  brigid-api          brigid-store          JtiStore (in-memory)
  │                         │                  (sqlite)                  │
  │── POST /auth/logout ──▶│                       │                     │
  │   Authorization:       │                       │                     │
  │   Bearer <jwt>         │ decode + verify token │                     │
  │                        │ blacklist_jti(jti,exp)│                     │
  │                        │───────── INSERT ────▶│ (durable, survives  │
  │                        │                       │  restart)           │
  │                        │ blacklist(jti, exp) ────────────────────▶│ TTL = exp
  │◄── 204 No Content ────│                       │                     │
  │                        │                       │                     │
  │── (any request with same token) ▶│            │                     │
  │                        │ AuthenticatedClaims:  │                     │
  │                        │  ① in-memory check ──────────────────────▶│
  │                        │  ② if not cached,     │                     │
  │                        │     SQL lookup ─────▶│ jti_blacklist row?  │
  │◄── 401 Unauthorized ──│                       │                     │
```

### Security properties

- The token is validated fully (signature, expiry, issuer, audience) before blacklisting.
- **Two-tier revocation store** — every logout writes to both:
  1. The durable SQLite `jti_blacklist` table in `brigid-store` (written **first**, so a crash between the two writes still revokes the token).
  2. The in-memory `JtiStore` cache in `brigid-oidc` (written second, for hot-path lookups without an SQL round-trip).
- `AuthenticatedClaims` consults **both** stores on every authenticated request: a process restart that drops the in-memory cache cannot resurrect a revoked token because the SQL row is authoritative.
- Both stores expire entries at the token's `exp` (TTL in memory, `WHERE expires_at > now` in SQL), so neither grows unbounded.
- `POST /auth/logout` is protected by the `AuthenticatedClaims` extractor: malformed or expired tokens are rejected before the handler runs.

*Reference:* [`brigid-api/src/routes/auth.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-api/src/routes/auth.rs), [`brigid-oidc/src/jti.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-oidc/src/jti.rs), [`brigid-store` JTI blacklist](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-store/src/lib.rs) — see also [`security-model.md` §4.4](./security-model.md#44-session-termination-logout) and [`audit-checklist.md` §2.2](./audit-checklist.md).

---

*Last updated: Phase 8 (audit readiness). Covers Phases 1–7.*
