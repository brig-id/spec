# Audit Checklist — brig·id

> **Audience:** external security auditors and advanced contributors.
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
  - *Check:* [`brigid-oidc/src/token.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-oidc/src/token.rs) — `validate_token()` verifies EdDSA signature
- [ ] Can an attacker replay a valid ID token?
  - *Check (single-use path):* [`brigid-oidc/src/token.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-oidc/src/token.rs) — `validate_token()` calls `JtiStore::check_and_insert()` so a second presentation of the same `jti` is rejected.
  - *Check (bearer path):* [`brigid-api/src/middleware.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-api/src/middleware.rs) — `AuthenticatedClaims` calls `decode_token()`; bearer tokens are reusable until `exp` or logout-driven blacklist entry, and the auditor must confirm `decode_token` does NOT insert into `JtiStore` (otherwise legitimate repeated requests would be locked out).
- [ ] Does the `sub` claim ever expose a username, alias, or raw DID?
  - *Check:* [`brigid-oidc/src/token.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-oidc/src/token.rs) — `Claims::sub` must equal `vsid.to_string()`
- [ ] Is a logged-out token (via `POST /auth/logout`) rejected on subsequent requests?
  - *Check:* [`brigid-oidc/src/jti.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-oidc/src/jti.rs) — `JtiStore::is_blacklisted()` checked by `AuthenticatedClaims` extractor
- [ ] Is the JTI blacklist bounded (no unbounded growth)?
  - *Check:* [`brigid-oidc/src/jti.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-oidc/src/jti.rs) — entries expire at token `exp`

### 2.3 Identity correlation
- [ ] Can two relying parties correlate users via the `sub` claim?
  - *Check:* [`brigid-identity/src/vsid.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-identity/src/vsid.rs) — VSID includes `client_id` in HKDF `info`
- [ ] Can VSID be derived from an alias?
  - *Check:* `brigid-identity` tests — `vsid_never_derived_from_alias`

### 2.4 Storage compromise
- [ ] Does a raw SQLite dump reveal any readable secrets?
  - *Check:* `brigid-store/src/store.rs` — all `insert_*` functions encrypt before write
- [ ] Is the nonce unique per ciphertext?
  - *Check:* `brigid-crypto/src/aes.rs` — nonce generated via `OsRng` per call

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
- [ ] Is TLS 1.2 rejected? / Is TLS 1.3 minimum enforced?
  - *Check:* [`server-leaf/src/main.rs`](https://github.com/brig-id/server-leaf/blob/3c87d7d660ad61a2a228515ff831c48450da47ba/src/main.rs) — `ServerConfig::builder_with_protocol_versions(&[&TLS13])`
- [ ] Is OpenSSL in the dependency tree?
  - *Check:* `cargo tree -i openssl-sys` — present as transitive dep of webauthn-rs-core (X.509 attestation parsing); must NOT be used for TLS (`cargo deny check bans` confirms)

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
- [ ] `jti` is always generated (UUID v4) and persisted by `brigid-store` on the
      single-use validation / logout paths (the in-process `JtiStore` is only
      the fast-path cache; bearer tokens validated against persisted claims are
      not inserted into `JtiStore` itself)
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
- [ ] Binary refuses to start if `BRIGID_MASTER_KEY` (or `BRIGID_MASTER_KEY_FILE`)
      is absent, is not a 64-character ASCII hex string, or does not decode to
      exactly 32 bytes (see `security-model.md` §6 and `operations.md` for the
      authoritative loader contract)
- [ ] `BRIGID_MASTER_KEY` does not appear in startup logs
- [ ] Container runs as non-root (`nonroot:nonroot`)
- [ ] Docker image is distroless (no shell, no package manager)
- [ ] Container filesystem is read-only (`read_only: true` in compose)
- [ ] `BRIGID_MASTER_KEY` supplied via Docker secret, not plain env var, in production

---

## 4. Automated Checks

### Required tooling

Install once before running the checks below (Rust ≥ 1.95 stable + nightly):

```bash
rustup toolchain install stable nightly
rustup target add wasm32-unknown-unknown            # for brigid-ui (Leptos)
cargo install --locked cargo-audit cargo-deny cargo-llvm-cov cargo-cyclonedx
cargo +nightly install --locked cargo-fuzz
# System linker required by the workspaces' `.cargo/config.toml`:
sudo apt-get install -y mold
```

The brig·id devcontainer (`brig-id/.dev/.devcontainer/`) provides all of the
above out of the box.

### Commands and working directories

Each command must be run from the listed repository checkout. Clone all
sibling repositories into the same parent directory (the layout the
`brig-id/.dev` workspace expects); the reusable workflows in
`brig-id/.github` invoke the same commands from each repo's CI.

```bash
# In brig-id/crypto:
cargo audit                          # known CVEs
cargo deny check                     # license + supply chain
cargo clippy --all-targets --all-features -- -D warnings
cargo llvm-cov --summary-only        # crypto target: 100% (Phase 1)
cargo +nightly fuzz run fuzz_decrypt -- -max_total_time=60

# In brig-id/core (Cargo workspace under crates/):
cargo audit
cargo deny check
cargo clippy --all-targets --all-features -- -D warnings
cargo llvm-cov --workspace --summary-only   # workspace target: ≥ 95%

# In brig-id/server-leaf:
cargo audit
cargo deny check
cargo clippy --all-targets -- -D warnings
cargo cyclonedx                      # generate CycloneDX SBOM
```

---

## 5. Coordinated Vulnerability Disclosure

See [`SECURITY.md`](./SECURITY.md) for the CVD process and contact details.
All security reports should be submitted via GitHub's private vulnerability reporting.

---

*Last updated: Phase 8 (audit readiness). Covers Phases 1–7.*
