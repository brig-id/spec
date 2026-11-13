# Security Model — brig·id

> **Audience:** external security auditors and advanced contributors.
> This document describes the threat model, attack surfaces, trust assumptions,
> and mitigations for brig·id.

---

## 1. System Overview

brig·id is a self-hosted identity provider (IdP) offering:

- **Passkey-only authentication** (WebAuthn / FIDO2 — no passwords)
- **OIDC ID Token issuance** with VSID as the stable `sub` claim
- **Encrypted storage** — sensitive fields are encrypted before reaching SQLite, with keys the
  server derives from its master key (this protects a copy of the database, not against the
  server's operator: see §5, "Decided, not built yet")
- **Post-quantum readiness** — hybrid classical+PQC key pairs available

---

## 2. Trust Boundaries

```
┌─────────────────────────────────────────────────────┐
│  Client browser / native app                        │
│  (untrusted — attacker may control)                 │
└──────────────┬──────────────────────────────────────┘
               │  HTTPS (TLS 1.2 or 1.3, see §4.1)
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
| `BRIGID_MASTER_KEY` / `BRIGID_MASTER_KEY_FILE` env var | **Root of trust** | Must be a 64-character ASCII hex string encoding 32 random bytes (single trailing newline tolerated when read via `BRIGID_MASTER_KEY_FILE`); loaded once at startup via env var or Docker secret file |
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
- **Reference deployment:** Caddy terminates TLS in front of `leaf`, which
  listens on plain HTTP inside the container network
  (`server-leaf/deploy/compose.yaml`). The accepted TLS versions are Caddy's
  defaults: TLS 1.2 and 1.3.
- **Built-in TLS (optional):** when `tls_cert`/`tls_key` are configured,
  `leaf` terminates TLS itself via
  `axum_server::tls_rustls::RustlsConfig::from_pem_file()`, with rustls'
  default protocol versions: TLS 1.2 and 1.3. The minimum is **TLS 1.2**, not
  1.3: nothing restricts the versions (no
  `builder_with_protocol_versions(&[&TLS13])` call).
  No OpenSSL-backed TLS connections either way; openssl-sys appears as a
  transitive dependency of `webauthn-rs-core` (via `webauthn-attestation-ca`)
  for X.509 attestation certificate parsing — not for network connections.
  - *Verify:* run `cargo deny check bans`; the `deny.toml` skip-tree exception for `openssl-sys` documents this path. `cargo tree -i openssl-sys` shows it present; `cargo deny check bans` confirms it is allowed only as an attestation-parsing dep, not for TLS.
  - *Implementation reference:* [`server-leaf/src/main.rs`](https://github.com/brig-id/server-leaf/blob/main/src/main.rs) — `RustlsConfig::from_pem_file()` call, `bind_rustls()`.
- **HSTS** enforced via `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`.

### 4.2 Authentication endpoint (`/auth/*`)
- **Mitigation:** rate limiting per client IP via `tower-governor`, in two
  separate quotas: the WebAuthn ceremonies (register, login, add-passkey
  finish), open to anyone, 20 req/min in bursts of 5; the routes that need a
  session (`/auth/session*`, `/auth/passkeys*`, `/account/*`), 60 req/min in
  bursts of 30. Behind a reverse proxy the client IP is read from
  `X-Forwarded-For`, which requires `security.trust_forwarded_for = true`;
  without it every request carries the proxy's address and all clients share
  the same quotas.
- **Mitigation:** WebAuthn RP ID strict (no wildcard); signature counter verified on every login.
- **Mitigation:** challenges are single-use and expire (TTL enforced in `brigid-webauthn`).

### 4.3 Token issuance
- **Today:** none. No endpoint issues an ID token, so none can be minted for a
  caller-chosen `client_id` ([ADR-005](decisions/005-account-sessions-and-rp-tokens.md) §3).
- **With the relying-party flow (ADR-005 §2):** registered clients and exact redirect URIs,
  Authorization Code + PKCE (`S256`), `state` and `nonce`; `sub` = the VSID of the identity
  the user chose (ADR-007), `exp` five minutes, unique `jti`, EdDSA signature with stored,
  rotatable keys.

### 4.4 Account sessions and sign-out
- **Threat:** a script on the page steals the session and uses it elsewhere, or keeps it
  alive indefinitely; another site makes the browser act on the account.
- **Mitigation:** the session is a server-side session behind a 256-bit random token in an
  `__Host-` cookie, `HttpOnly` (scripts can't read it), `Secure`, `SameSite=Strict`. Only a
  SHA3-256 hash of the token is stored: a database copy gives no usable session.
- **Mitigation:** idle timeout (1 h, extended by use) and absolute lifetime (12 h, never
  extended): an XSS can act while the page is open, but can't take the session away or keep
  it alive past the lifetime.
- **Mitigation:** every request that changes something must carry this server's `Origin`
  (in addition to `SameSite=Strict`); account endpoints take the account from the session,
  never from a parameter, so no request can name another account.
- **Mitigation:** sign-out deletes the session server-side; the cookie is cleared too.
- *Verify:* core integration test `cookie_session_check_sign_out_and_same_origin`; store
  test `sessions_expire_idle_and_absolute_and_end_on_sign_out`.

### 4.5 Storage (zero-trust)
- **Mitigation:** `brigid-store` encrypts every sensitive field with AES-256-GCM + unique nonce before `INSERT`.
- **Mitigation:** `MASTER_KEY` is never written to disk; loaded from env var or Docker secret.
- **Mitigation:** HKDF-SHA3-256 used to derive per-context sub-keys from the master key.

### 4.6 Cross-site scripting (XSS)
- **Mitigation:** `Content-Security-Policy` header restricts `default-src`,
  `frame-ancestors`, `object-src`, and `base-uri`. Two policies exist, by
  design:
  - API responses (JSON only) get the strict policy of
    `brigid-api/src/router.rs` (`build_router()`), with no inline source.
  - UI responses (`/login`, `/register`, static assets, …) get the policy of
    [`server-leaf/src/lib.rs`](https://github.com/brig-id/server-leaf/blob/main/src/lib.rs)'s
    `apply_ui_fallback()`. It is applied after the static-file fallback is
    attached, because in axum 0.8 a `.layer()` only wraps the routes that
    exist when it is called: the layer inside `build_router()` cannot reach
    a fallback added later.
- **Inline scripts and styles by hash:** the Qwik static (SSG) build emits
  inline `<script>`/`<style>` content (its resumability bootstrap), and with
  no per-request rendering step a nonce can't reach it. Instead `app`'s build
  writes `dist/csp-hashes.json`, the `'sha256-…'` source of every inline block,
  and `server-leaf` puts exactly those in `script-src`/`style-src` of UI
  responses ([`brig-id/app#3`](https://github.com/brig-id/app/issues/3),
  [`brig-id/core#20`](https://github.com/brig-id/core/issues/20)). Only
  well-formed hash sources are copied from the manifest. The hashes change from
  one build to the next, so the manifest ships inside each `dist/`.
- **Residual deviation:** a UI build without the manifest falls back to
  `'unsafe-inline'` (logged as a warning at startup). `style-src`/`font-src`
  allow the third-party font host `https://fonts.bunny.net`; `img-src` and
  `connect-src` allow Unsplash (the login page's background photo).
- *Verify:* `server-leaf` unit tests `hashes_replace_unsafe_inline` and
  `without_a_manifest_falls_back_to_unsafe_inline` (`src/lib.rs`).

### 4.7 Cross-origin requests (CORS)
- **Mitigation:** explicit origin allowlist in configuration; no `Access-Control-Allow-Origin: *`.
- **Mitigation:** allowed headers enumerated (`Content-Type`, `Authorization`) — no wildcard `Access-Control-Allow-Headers`.

---

## 5. Security Invariants

These invariants are enforced in code and must be verified by auditors:

1. **VSID stability:** same `(identity, client_id)` → same VSID (the VSID secret is a single deployment-wide value, stored encrypted, not a per-call input); different `client_id` → different VSID.
   - *Reference:* [`brigid-identity/src/vsid.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-identity/src/vsid.rs) — `identity_vsid()`
   - *Verify:* tests `deterministic`, `different_client_different_vsid`, `known_answer` in the same file.
2. **VSID isolation:** the VSID is derived from an identity's UUID only, never from an account, a username, a display name or a DID ([ADR-007](./decisions/007-identifiers.md)); two identities of one account have different VSIDs at the same relying party.
   - *Verify:* test `different_identity_different_vsid`; `identity_vsid()` takes a `Uuid`, so no other input can be passed.
3. **Zero raw secrets in storage:** every *sensitive* field written by `brigid-store` (credential bodies, signing-key seeds, user-supplied identifiers, etc.) MUST be encrypted with AES-256-GCM via `brigid-crypto` before the `INSERT` reaches SQLite. Non-secret *operational metadata* — table primary keys, foreign keys, `expires_at` timestamps, schema-migration markers, and the session table's token **hashes** (SHA3-256 of a 256-bit random token: a hash is not a usable session) — may be stored in plaintext so the database can be queried and pruned. What MUST never appear in plaintext is any value that would reveal a credential, a private key, a user-supplied identifier, or the contents of an active session.
   - *Reference:* [`brigid-store/src/store.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-store/src/store.rs) — see the `encrypt_field` / `decrypt_field` helpers and their call sites in every public store function.
   - *Verify:* `dump_contains_no_plaintext` integration test in `brigid-store` — it asserts the secret payloads do not appear in a raw `.dump`, but tolerates plaintext metadata columns by construction.
4. **No secrets in logs:** `tracing` spans must not capture key material, WebAuthn private keys, or OIDC signing keys.
5. **No token as an account credential:** brig·id's API authenticates the account by its
   server-side session cookie only ([ADR-005](./decisions/005-account-sessions-and-rp-tokens.md));
   it accepts no ID token, so there is nothing to replay against it. ID tokens, once the
   relying-party flow exists, are for relying parties, single audience, five-minute lifetime.
   - *Reference:* [`brigid-api/src/session.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-api/src/session.rs) — `AccountSession`, `require_same_origin`.
6. **Key zeroization:** `OidcSigningKey` inner `SigningKey` (ed25519-dalek) implements `ZeroizeOnDrop`.

---

### Decided, not built yet

These hold by decision and are not enforced by the code described here. Each joins the list
above, with its reference and its test, when it is implemented.

- **No link across servers:** a server holds no reference to an account or an identity of
  another server ([ADR-021](./decisions/021-sharing-between-servers.md)).
- **Labels never leave:** an identity's name and logo are never sent to a relying party,
  which receives a VSID and the attributes the person accepted to share
  ([ADR-022](./decisions/022-identity-attributes-and-consent.md)).
- **What describes a person is unreadable to the operator:** labels and attributes are
  encrypted end to end with a key only a passkey gives, and only passkeys that can give one
  are accepted ([ADR-023](./decisions/023-end-to-end-encryption.md), with what it does not
  protect against).
- **No way in for an administrator:** no administrator can give access back to an account,
  and a recovery code is mandatory
  ([ADR-018](./decisions/018-app-structure.md), [ADR-024](./decisions/024-account-lifecycle.md)).

---

## 6. Key Material Lifecycle

```
Startup: BRIGID_MASTER_KEY (env var)  OR  BRIGID_MASTER_KEY_FILE (Docker/Swarm/K8s secret file)
         ↓
         loader validates 64-char ASCII hex, decodes to 32 bytes
         ↓
         HKDF derives per-context keys (per-account storage keys, the keys
         that encrypt the stored OIDC signing seeds and the VSID secret)
         ↓
         OIDC signing keys and the VSID secret are read from the database
         and decrypted
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

Summary: `brigid-crypto` provides the X-Wing hybrid KEM (ML-KEM-768 + X25519) and composite
ML-DSA-65 + Ed25519 signatures, both following their IETF drafts and checked against the
drafts' test vectors. **Nothing in brig·id uses them yet**: OIDC tokens are signed with
Ed25519, and no data is encrypted to a KEM key. They are ready for when a use case needs them.

---

## 8. Out of Scope

- Host OS security (hardening the server running brig·id)
- Browser/client security beyond CSP and HTTPS
- Certificate authority / PKI management (operator responsibility)
- Network-level DDoS mitigation (operator responsibility — put behind a reverse proxy/CDN)
