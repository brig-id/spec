# Security Model — brig·id

> **Audience:** external security auditors and advanced contributors.
> This document describes the threat model, attack surfaces, trust assumptions,
> and mitigations for brig·id.

---

## 1. System Overview

brig·id is a self-hosted identity provider (IdP) offering:

- **Passkey-only authentication** (WebAuthn / FIDO2 — no passwords)
- **OIDC ID Token issuance** with VSID as the stable `sub` claim
- **Zero-trust storage** — all sensitive data encrypted before reaching SQLite
- **Post-quantum readiness** — hybrid classical+PQC key pairs available

---

## 2. Trust Boundaries

```
┌─────────────────────────────────────────────────────┐
│  Client browser / native app                        │
│  (untrusted — attacker may control)                 │
└──────────────┬──────────────────────────────────────┘
               │  HTTPS (TLS 1.3 only)
┌──────────────▼──────────────────────────────────────┐
│  brigid-api  (Axum HTTP server)                     │
│  ─ rate limiting (tower-governor)                   │
│  ─ CSP + security headers                          │
│  ─ input validation via serde                       │
└────┬────────────────────┬───────────────────────────┘
     │                    │
┌────▼────┐          ┌────▼────────────────────────────┐
│ brigid- │          │ brigid-store (SQLite)           │
│ oidc    │          │ Sensitive fields encrypted      │
│ (JWT)   │          │ (AES-256-GCM) before INSERT;    │
│         │          │ row metadata (created_at,       │
│         │          │ HKDF-SHA3-256 username_index)   │
│         │          │ stored in plaintext for query   │
└─────────┘          └─────────────────────────────────┘
```

### Trust levels

| Component | Trust level | Notes |
|---|---|---|
| `BRIGID_MASTER_KEY` / `BRIGID_MASTER_KEY_FILE` env var | **Root of trust** | Must be 32 random bytes; loaded once at startup via env var or Docker secret file |
| TLS certificate | High | Controls transport security |
| SQLite database file | Low (zero-trust) | Raw dump must reveal no readable secrets |
| Client browser | Untrusted | |
| Relying party (RP) app | Semi-trusted | Registered `client_id` only |

---

## 3. Threat Actors

| Actor | Capability | Goal |
|---|---|---|
| Network attacker (passive) | TLS interception attempt | Steal tokens / credentials |
| Network attacker (active MITM) | TLS stripping | Steal tokens |
| Compromised database | Read SQLite file | Recover user identities or credentials |
| Compromised process memory | Read heap | Recover key material |
| Malicious RP | Token replay, VSID correlation | Link identities across services |
| Malicious user | Brute force, replay | Authenticate as another user |
| XSS attacker | Inject scripts into UI | Steal session tokens |

---

## 4. Attack Surfaces

### 4.1 Transport layer
- **Mitigation:** TLS 1.3 minimum, configured via rustls `ServerConfig::builder_with_protocol_versions(&[&TLS13])`. No OpenSSL-backed TLS connections; openssl-sys appears as a transitive dependency of `webauthn-rs-core` (via `webauthn-attestation-ca`) for X.509 attestation certificate parsing — not for network connections.
  - *Verify:* run `cargo deny check bans`; the `deny.toml` skip-tree exception for `openssl-sys` documents this path. `cargo tree -i openssl-sys` shows it present; `cargo deny check bans` confirms it is allowed only as an attestation-parsing dep, not for TLS.
  - *Implementation reference:* [`server-leaf/src/main.rs`](https://github.com/brig-id/server-leaf/blob/dev/src/main.rs) — `rustls::ServerConfig` construction.
- **HSTS** enforced via `Strict-Transport-Security: max-age=31536000; includeSubDomains`.

### 4.2 Authentication endpoint (`/auth/*`)
- **Mitigation:** rate limiting — 20 req/min per IP via `tower-governor`.
- **Mitigation:** WebAuthn RP ID strict (no wildcard); signature counter verified on every login.
- **Mitigation:** challenges are single-use and expire (TTL enforced in `brigid-webauthn`).

### 4.3 Token issuance
- **Mitigation:** `sub` claim = VSID (derived from `(did_root, client_id, salt)`) — stable and opaque per RP.
- **Mitigation:** `jti` replay prevention in `JtiStore` (TTL = token `exp`); store evicts expired entries.
- **Mitigation:** EdDSA signatures (Ed25519) over ID tokens; private key never leaves the server process.

### 4.4 Session termination (logout)
- **Endpoint:** `POST /auth/logout` — requires `Authorization: Bearer <token>`.
- **Mitigation:** on logout, the token's `jti` is inserted into `JtiStore::blacklist()`. Subsequent calls with the same token are rejected by `is_blacklisted()` before any handler runs.
- **Mitigation:** blacklisted entries expire automatically at the token's `exp` timestamp — the JTI store is always bounded.
- *Reference:* [`brigid-oidc/src/jti.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/jti.rs) — `blacklist()` / `is_blacklisted()`; [`brigid-api/src/routes/auth.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-api/src/routes/auth.rs).

### 4.5 Storage (zero-trust)
- **Mitigation:** `brigid-store` encrypts every sensitive field with AES-256-GCM + unique nonce before `INSERT`.
- **Mitigation:** `MASTER_KEY` is never written to disk; loaded from env var or Docker secret.
- **Mitigation:** HKDF-SHA3-256 used to derive per-context sub-keys from the master key.

### 4.6 Cross-site scripting (XSS)
- **Mitigation:** `Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{…}'; …` — no `unsafe-inline`.
- **Mitigation:** nonce regenerated per request; injected into Leptos hydration `<script>` tags.

### 4.7 Cross-origin requests (CORS)
- **Mitigation:** explicit origin allowlist in configuration; no `Access-Control-Allow-Origin: *`.
- **Mitigation:** allowed headers enumerated (`Content-Type`, `Authorization`) — no wildcard `Access-Control-Allow-Headers`.

---

## 5. Security Invariants

These invariants are enforced in code and must be verified by auditors:

1. **VSID stability:** same `(did_root, client_id, salt)` → same VSID; different `client_id` → different VSID.
   - *Reference:* [`brigid-identity/src/vsid.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-identity/src/vsid.rs) — `compute_vsid()`
   - *Verify:* `vsid_is_stable` and `vsid_non_correlable_across_clients` tests in the same file.
2. **VSID isolation:** VSID must never be derived from an alias or a virtual identity.
   - *Reference:* [`brigid-identity/src/root_id.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-identity/src/root_id.rs) — `RootId::parse()` validates format.
   - *Verify:* `vsid_never_derived_from_alias` test asserts `compute_vsid` rejects non-`did:web:` inputs with an error.
3. **Zero raw secrets in storage:** `brigid-store` must never call SQLite `execute`/`fetch` without encrypted payloads.
   - *Reference:* [`brigid-store/src/store.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-store/src/store.rs) — all public store functions.
   - *Verify:* `dump_contains_no_plaintext` integration test in `brigid-store`.
4. **No secrets in logs:** `tracing` spans must not capture key material, WebAuthn private keys, or OIDC signing keys.
5. **Single-use JWTs:** `jti` must be checked in `JtiStore` before a token is accepted.
   - *Reference:* [`brigid-oidc/src/jti.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/jti.rs) — `check_and_insert()`.
   - *Verify:* `replayed_jti_is_rejected` test in the same file.
6. **Key zeroization:** `OidcSigningKey` inner `SigningKey` (ed25519-dalek) implements `ZeroizeOnDrop`.

---

## 6. Key Material Lifecycle

```
Startup: BRIGID_MASTER_KEY loaded from env
         ↓
         HKDF derives per-context keys (storage, token signing)
         ↓
Runtime: keys held in process memory only
         ↓
Shutdown: keys dropped (zeroized on drop via secrecy/zeroize)
```

### MASTER_KEY requirements
- 32 cryptographically random bytes (256 bits)
- Encoded as 64 hex characters in the `BRIGID_MASTER_KEY` env var, or stored as a Docker secret file referenced by `BRIGID_MASTER_KEY_FILE`
- Must never appear in logs, config files, error messages, or core dumps

---

## 7. Post-Quantum Readiness

See [`pqc.md`](pqc.md) for the full PQC strategy and migration plan.

Summary: hybrid ML-KEM-768+X25519 (FIPS 203) and ML-DSA-65+Ed25519 (FIPS 204) are available in
`brigid-crypto`. Current deployment uses classical Ed25519 for OIDC token signing;
PQC hybrid signatures will be activated in a future phase.

---

## 8. Out of Scope

- Host OS security (hardening the server running brig·id)
- Browser/client security beyond CSP and HTTPS
- Certificate authority / PKI management (operator responsibility)
- Network-level DDoS mitigation (operator responsibility — put behind a reverse proxy/CDN)

---

*Last updated: Phase 8 (audit readiness). Covers Phases 1–7.*
