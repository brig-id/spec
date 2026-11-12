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
| PQC KEM | X-Wing: ML-KEM-768 + X25519 | 1216-byte public key, 32-byte seed | FIPS 203, draft-connolly-cfrg-xwing-kem |
| PQC signatures | Composite ML-DSA-65 + Ed25519 (SHA-512 pre-hash) | 1984-byte public key, 3373-byte signature | FIPS 204, draft-ietf-lamps-pq-composite-sigs |

---

## 2. VSID Computation

VSID (Virtual Stable Identifier) is the stable, opaque `sub` claim in OIDC tokens.

> **Accepted, not implemented:** [ADR-007](./decisions/007-identifiers.md) replaces the
> identity DID in the formula below by the identity's UUID (`info = "brigid-vsid-v2" ||
> identity_uuid || u32_be(len(client_id)) || client_id`) and removes account and identity
> DIDs. The rest of this section describes the code as it runs today.
It ensures that the same user has a **different identifier per relying party**,
preventing cross-RP identity correlation.

### Formula

The keying material is the **VSID secret**: one value per deployment, the same
for every identity. It is 32 random bytes generated when the database is
created and stored in it, encrypted with a key derived from `MASTER_KEY`
([ADR-006](./decisions/006-vsid-secret.md)).

A database created before the secret was stored keeps the value it used until
then, `HKDF-SHA3-256(ikm = MASTER_KEY, info = "brigid-vsid-salt")[..32]`,
recorded as its secret at the first start or the first `rotate-key` of the
upgraded server: its VSIDs do not change.

```
VSID = HKDF-SHA3-256(
    ikm  = vsid_secret,                   // deployment-wide, stored encrypted
    salt = none,                          // HKDF's own salt param is unused (zero-padded)
    info = "brigid-vsid-v1" ||
           u32_be(len(did_root)) || did_root ||
           u32_be(len(client_id)) || client_id
)[..32]
→ encoded as unpadded base64url (43 chars)
```

Length-prefixing `did_root` and `client_id` in `info` prevents collisions
from the `:` characters that appear in both DIDs and, potentially, client
IDs.

### Invariants

1. Same `(did_root, client_id)` → same VSID (deterministic; the VSID secret
   is constant per deployment, not a per-call input).
2. Different `client_id` → different VSID (unlinkable across RPs).
3. VSID must be derived from an **identity DID** only, never from an account DID, an
   alias, a display name or a username (see [ADR-002](./decisions/002-accounts-and-identities.md)).
   *Implementation status:* no token is issued today; the account has a server-side session,
   not a token ([ADR-005](./decisions/005-account-sessions-and-rp-tokens.md)). With
   [ADR-007](./decisions/007-identifiers.md) the input is the identity's UUID.
4. VSID must **never** be derived from a username directly.
5. Rotating `MASTER_KEY` re-encrypts the VSID secret and changes no VSID —
   see the rotation runbook in [`operations.md`](./operations.md). The VSID
   still depends on the server's domain, through the identity DID.

*Reference implementation:* [`brigid-identity/src/vsid.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-identity/src/vsid.rs) — `compute_vsid()`, `derive_vsid_salt()` (upgrade path only);
[`brigid-store/src/store.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-store/src/store.rs) — `vsid_secret()`

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
- The server part of the username must be this server's domain: an account cannot be
  registered under a domain the server does not control.
- Attestation is not requested (`attestation: "none"`, the `webauthn-rs` passkey
  profile): the server learns no authenticator make or model, and does not restrict
  which authenticators may register.

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
  │                                 │ open session (hash only) ────▶│
  │◀── 204, Set-Cookie: __Host-brigid_session=… ──────────────────│
```

Signing in opens an **account session** ([ADR-005](./decisions/005-account-sessions-and-rp-tokens.md)
§1), it issues no token: brig·id's own API never takes an ID token as a credential.
Signup (`/auth/register/finish`) and recovery (`/auth/recovery/finish`) open one the same way.

### Username-less login (discoverable credentials)

`POST /auth/login/begin` without `username` starts a discoverable
(conditional-UI) challenge with an empty `allowCredentials`. At finish, the
user is taken from the assertion's `userHandle` (the account id bound at
registration) and the assertion is verified against **that user's passkeys
only**: the handle is a claim until the signature checks. An unknown handle
answers `401`.

### Security checks (authentication)
- Challenge must be single-use (deleted after verification).
- Signature counter: when the authenticator maintains one, it must be greater than the
  stored value (cloned-authenticator detection). Synced passkeys report a counter of `0`
  on every assertion; `webauthn-rs` accepts `0` when the stored value is also `0`.
- RP ID verified against server domain.

---

## 5. OIDC ID Token

> **Status ([ADR-005](./decisions/005-account-sessions-and-rp-tokens.md)):** no endpoint
> issues an ID token today. Tokens come back with the Authorization Code flow of §2 of that
> ADR, for an identity chosen by the user, with `sub` = the identity's VSID (ADR-007). The
> structure below is what `brigid-oidc` issues.

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
  "server":     "example.com",       // issuing server hostname, from its configuration
  "alias_type": "public"             // constant; to be removed (ADR-002)
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

`brigid-oidc` still has `validate_token` / `decode_token` and a `JtiStore`, from when the
account session was a bearer ID token. **Nothing in the API calls them since ADR-005:**
brig·id's API takes no ID token as a credential. A relying party validates the tokens it
receives (signature with the JWKS, `iss`, `aud`, `exp`, `nonce`) with its own OIDC library.
Removing the unused validation code from `brigid-oidc` is a follow-up.

---

### Account session (ADR-005 §1)

| Property | Value |
|---|---|
| Cookie | `__Host-brigid_session=<256 random bits, base64url>; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=<lifetime>` (over plain-HTTP development: `brigid_session`, without `Secure`) |
| Stored | SHA3-256 of the token, account id, idle deadline, absolute end (table `sessions`) |
| Idle timeout | 1 h (`security.session_ttl_seconds`), pushed back by every request, never past the end |
| Absolute lifetime | 12 h after the passkey ceremony (`security.session_lifetime_seconds`), never extended |
| Account endpoints | take the account from the session; no endpoint accepts a `user_id` |
| Cross-site requests | any request but GET/HEAD/OPTIONS must carry this server's `Origin` (403 otherwise) |

| Endpoint | Effect |
|---|---|
| `GET /auth/session` | `200 { idle_expires_at, expires_at }`, or `401` when there is no live session or its account is gone |
| `POST /auth/logout` | deletes the session, clears the cookie (`204`) |

---

## 6. Discovery Endpoints

### `GET /.well-known/openid-configuration`

**Not served** until the relying-party flow of [ADR-005](./decisions/005-account-sessions-and-rp-tokens.md)
§2 (Authorization Code + PKCE, registered clients) exists: a discovery document that no
client can use is worse than none (§3). The JWKS and the DID document stay.

### `GET /.well-known/jwks.json`

JWK Set containing the Ed25519 public keys that verify tokens: the current signing
key first, then any key retired by a rotation and still within its grace period
(see [`operations.md`](./operations.md) §2). A token names its key in the `kid` header;
a token without `kid`, or with an unknown one, is rejected.

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

**Important implementation note:** `jsonwebtoken`'s `DecodingKey::from_ed_der` must be
given the raw 32-byte public key, **not** SubjectPublicKeyInfo DER
([ADR-001](./decisions/001-ring-eddsa-decoding-key-format.md)). See [`brigid-oidc/src/key.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-oidc/src/key.rs) — `decoding_key()`.

---

## 7. DID Web Document

### `GET /.well-known/did.json`

Exposes the server's DID Web document, enabling DID-based identity resolution.

*Reference:* [`brigid-did/src/web.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-did/src/web.rs) — `did_web_document()`

---

## 8. Logout Flow

`POST /auth/logout` (same-origin, with the session cookie) deletes the session row and
answers `204` with a `Set-Cookie` that removes the cookie. The session cannot be revived:
its token is unknown from then on, across restarts too. There is no token to revoke, so no
JTI blacklist.

*Reference:* [`brigid-api/src/session.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-api/src/session.rs),
`EncryptedStore::{create_session, use_session, delete_session}`.
