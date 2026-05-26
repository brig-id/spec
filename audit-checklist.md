# Audit Checklist — brig·id

> **Audience:** third-party security auditors.
> Use this checklist to guide the security review of brig·id.
> Each item references the relevant source file and function.

---

## 1. Scope

This audit covers:

| Crate | Phase | Purpose |
|---|---|---|
| `brigid-crypto` | 1 | Cryptographic primitives |
| `brigid-store` | 2 | Zero-trust SQLite storage |
| `brigid-did` | 2 | DID:web and DID:peer resolution |
| `brigid-identity` | 3 | VSID computation, `RootId` |
| `brigid-webauthn` | 4 | Passkey registration and authentication |
| `brigid-oidc` | 5 | JWT issuance and validation |
| `brigid-api` | 6 | Axum HTTP server |
| `brigid-ui` | 6 | Leptos SSR frontend |

---

## 2. Critical Attack Vectors

### 2.1 Authentication bypass
- [ ] Can an attacker forge a WebAuthn assertion without the private key?
  - *Check:* `brigid-webauthn/src/` — assertion verification; signature verified via `webauthn-rs`
- [ ] Can an attacker replay a valid WebAuthn assertion?
  - *Check:* signature counter strictly increasing; challenge single-use
- [ ] Can an attacker bypass rate limiting on `/auth/*`?
  - *Check:* `brigid-api/src/` — `tower-governor` configuration

### 2.2 Token forgery
- [ ] Can an attacker forge an ID token without the server's Ed25519 private key?
  - *Check:* [`brigid-oidc/src/token.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/token.rs) — `validate_token()` verifies EdDSA signature
- [ ] Can an attacker replay a valid ID token?
  - *Check:* [`brigid-oidc/src/jti.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/jti.rs) — `JtiStore::check_and_insert()` prevents replay
- [ ] Does the `sub` claim ever expose a username, alias, or raw DID?
  - *Check:* [`brigid-oidc/src/token.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/token.rs) — `Claims::sub` must equal `vsid.to_string()`
- [ ] Is a logged-out token (via `POST /auth/logout`) rejected on subsequent requests?
  - *Check:* [`brigid-oidc/src/jti.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/jti.rs) — `JtiStore::is_blacklisted()` checked by `AuthenticatedClaims` extractor
- [ ] Is the JTI blacklist bounded (no unbounded growth)?
  - *Check:* [`brigid-oidc/src/jti.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-oidc/src/jti.rs) — entries expire at token `exp`

### 2.3 Identity correlation
- [ ] Can two relying parties correlate users via the `sub` claim?
  - *Check:* [`brigid-identity/src/vsid.rs`](https://github.com/brig-id/core/blob/dev/crates/brigid-identity/src/vsid.rs) — VSID includes `client_id` in HKDF `info`
- [ ] Can VSID be derived from an alias?
  - *Check:* `brigid-identity` tests — `vsid_is_not_derived_from_alias`

### 2.4 Storage compromise
- [ ] Does a raw SQLite dump reveal any readable secrets?
  - *Check:* `brigid-store/src/store.rs` — all `insert_*` functions encrypt before write
- [ ] Is the nonce unique per ciphertext?
  - *Check:* `brigid-crypto/src/aes_gcm.rs` — nonce generated via `OsRng` per call

### 2.5 Key material exposure
- [ ] Can the `BRIGID_MASTER_KEY` appear in logs, errors, or panics?
  - *Check:* grep for `BRIGID_MASTER_KEY` in non-config code — must be absent
- [ ] Is key material zeroized on drop?
  - *Check:* `brigid-crypto` — `Secret<T>` wrapper, `ZeroizeOnDrop` on `SigningKey`

### 2.6 XSS and injection
- [ ] Does the server emit a strict `Content-Security-Policy` header?
  - *Check:* `brigid-api/src/middleware/csp.rs`
- [ ] Does the CSP include `unsafe-inline` for scripts?
  - *Check:* must be absent; nonce-based for hydration scripts
- [ ] Can an attacker inject HTML into server-rendered pages?
  - *Check:* Leptos escapes all dynamic values by default

### 2.7 Transport security
- [ ] Is TLS 1.2 accepted?
  - *Check:* [`server-leaf/src/main.rs`](https://github.com/brig-id/server-leaf/blob/dev/src/main.rs) — `ServerConfig::builder_with_protocol_versions(&[&TLS13])`
- [ ] Is OpenSSL in the dependency tree?
  - *Check:* `cargo tree | grep openssl` — must return empty

---

## 3. Code Review Checklist

### Cryptography
- [ ] No hardcoded keys or test vectors in non-test code
- [ ] All `unwrap()` calls on crypto paths are justified or replaced with `?`
- [ ] Nonces generated via `OsRng`, never deterministic or counter-based
- [ ] HKDF `info` field is set for every derived key (domain separation)

### WebAuthn
- [ ] RP ID is strict (exact match to `domain` config, no suffix matching)
- [ ] Challenge TTL is enforced (challenges expire)
- [ ] Attestation verification is not disabled in production config

### OIDC
- [ ] `exp` is always set and checked
- [ ] `jti` is always generated (UUID v4) and stored in `JtiStore`
- [ ] `JtiStore` evicts expired entries (bounded growth)
- [ ] `aud` is verified against `expected_client_id`

### HTTP
- [ ] All `/auth/*` routes are behind the rate limiter
- [ ] CORS `allow_origin` is explicit (no `Any`)
- [ ] `X-Content-Type-Options: nosniff` is set
- [ ] `X-Frame-Options: DENY` is set
- [ ] `Strict-Transport-Security` is set
- [ ] `POST /auth/logout` requires a valid `Authorization: Bearer` header
- [ ] Logged-out tokens are rejected via `AuthenticatedClaims` before reaching handlers

### Deployment (server-leaf)
- [ ] Binary refuses to start if `BRIGID_MASTER_KEY` is absent or < 32 bytes
- [ ] `BRIGID_MASTER_KEY` does not appear in startup logs
- [ ] Container runs as non-root (`nonroot:nonroot`)
- [ ] Docker image is distroless (no shell, no package manager)
- [ ] Container filesystem is read-only (`read_only: true` in compose)
- [ ] `BRIGID_MASTER_KEY` supplied via Docker secret, not plain env var, in production

---

## 4. Automated Checks

Run the following before auditing:

```bash
cargo audit                          # known CVEs
cargo deny check                     # license + supply chain
cargo clippy --all-targets --all-features -- -D warnings
cargo llvm-cov --workspace --summary-only  # must be ≥ 95% (100% on crypto)
cargo cyclonedx                      # generate SBOM
cargo +nightly fuzz run fuzz_decrypt -- -max_total_time=60  # in brigid-crypto
```

---

## 5. Coordinated Vulnerability Disclosure

See [`SECURITY.md`](./SECURITY.md) for the CVD process and contact details.
All security reports should be submitted via GitHub's private vulnerability reporting.

---

*Last updated: Phase 8 (audit readiness). Covers Phases 1–7.*
