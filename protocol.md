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
| PQC KEM | ML-KEM-768 + X25519 (hybrid) | 1184-byte public key | FIPS 203 |
| PQC signatures | ML-DSA-65 + Ed25519 (hybrid) | 1952-byte public key | FIPS 204 |

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

*Reference implementation:* [`brigid-identity/src/vsid.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-identity/src/vsid.rs) — `compute_vsid()`

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
  "did":        "did:web:example.com",
  "server":     "example.com",
  "alias_type": "root"
}
```

### Token validation (brigid-oidc)

1. Verify EdDSA signature using the server's `OidcSigningKey`.
2. Check `exp` > now.
3. Check `iss` == expected issuer.
4. Check `aud` == expected `client_id`.
5. Check `jti` not in `JtiStore` (anti-replay); insert it with TTL = `exp`.

*Reference:* [`brigid-oidc/src/token.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/token.rs) — `validate_token()`

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
SubjectPublicKeyInfo DER. See [`brigid-oidc/src/key.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/key.rs) — `decoding_key()`.

---

## 7. DID Web Document

### `GET /.well-known/did.json`

Exposes the server's DID Web document, enabling DID-based identity resolution.

*Reference:* [`brigid-did/src/web.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-did/src/web.rs) — `did_web_document()`

---

## 8. Logout Flow

```
Client                          brigid-api                  JtiStore (in-memory)
  │                                 │                               │
  │─── POST /auth/logout ────────▶│                               │
  │    Authorization: Bearer <jwt>  │                               │
  │                                 │ decode + verify token         │
  │                                 │ check is_blacklisted(jti) ──▶│
  │                                 │ blacklist(jti, exp) ────────▶│ insert jti, TTL = exp
  │◄── 204 No Content ────────────│                               │
  │                                 │                               │
  │─── (any request with same token)▶│                               │
  │                                 │ check is_blacklisted(jti) ──▶│
  │                                 │◄─────────────────── true ────│
  │◄── 401 Unauthorized ───────────│                               │
```

### Security properties
- The token is validated fully (signature, expiry, issuer, audience) before blacklisting.
- Blacklisted JTIs expire automatically at the token's `exp` — `JtiStore` is always bounded.
- `POST /auth/logout` is protected by the `AuthenticatedClaims` extractor: malformed or expired tokens are rejected before the handler runs.

*Reference:* [`brigid-api/src/routes/auth.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-api/src/routes/auth.rs), [`brigid-oidc/src/jti.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/jti.rs)

---

*Last updated: Phase 8 (audit readiness). Covers Phases 1–7.*
