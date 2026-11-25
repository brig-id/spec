# Security Model — brig·id

> **Audience:** external security auditors and advanced contributors.
> Threats, trust boundaries, attack surfaces, invariants, the authentication policy and
> end-to-end encryption. Statuses are explained in the [README](README.md).

---

## 1. System Overview

brig·id is a self-hosted identity provider:

- **Passkeys only** (WebAuthn): no password, no other way in (§5)
- **An ID token per application**, whose subject is a VSID: one identifier per identity and
  per relying party ([`protocol.md`](protocol.md) §2, §6)
- **Encrypted storage**: sensitive fields are encrypted before reaching SQLite, with keys the
  server derives from its master key. This protects a copy of the database, not against the
  server's operator; that is what end-to-end encryption is for (§7)
- **Post-quantum readiness**: hybrid primitives available ([`pqc.md`](pqc.md))

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
| Relying party (RP) app | Semi-trusted | A registered `client_id`, or the URL of the document it publishes (`protocol.md` §6) |

---

## 3. Threat Actors

| Actor | Capability | Goal |
|---|---|---|
| Network attacker (passive) | TLS interception attempt | Steal tokens / credentials |
| Network attacker (active MITM) | TLS stripping | Steal tokens |
| Compromised database | Read SQLite file | Recover user identities or credentials |
| Compromised process memory | Read heap | Recover key material |
| Malicious RP | Token replay, VSID correlation, asking for more than it needs | Link identities across services |
| Server operator | Reads the database and holds the master key; serves the interface | Read what describes a person; learn where identities are used (§7) |
| Server administrator | Uses the administration pages | Enter someone's account, read their identities ([`model.md`](model.md) §7) |
| Malicious user | Brute force, replay | Authenticate as another user |
| XSS attacker | Inject scripts into UI | Steal session tokens |

---

## 4. Attack Surfaces

### 4.1 Transport layer

The server holds its certificate in one of three ways, chosen in its
configuration; naming half of one, or two at once, stops it at start.

- **ACME (`server.acme = true`), the server alone on port 443.** It obtains a
  certificate for `server.domain` from an ACME authority (RFC 8555; Let's
  Encrypt unless `server.acme_directory` names another) and renews it once two
  thirds of its lifetime have passed. The domain is proved with TLS-ALPN-01
  (RFC 8737) on the TLS listener itself: a handshake that offers only the
  `acme-tls/1` protocol receives a self-signed certificate carrying the SHA-256
  digest of the key authorization, every other handshake the current
  certificate. No other port is opened. A domain that is an IP address, a single
  label or under `.localhost` is refused at start.
  **Status:** built (core#122).
- **Certificate files (`tls_cert`/`tls_key`).** The operator obtains and renews
  them; a file that cannot be read stops the server before it listens.
  **Status:** built.
- **Behind a proxy (neither).** Plain HTTP on the internal network; the proxy
  (Caddy, nginx) holds the certificate and its TLS versions are its own.
  **Status:** built.

**The built-in listener speaks TLS 1.3 only** (`builder_with_protocol_versions(&[&TLS13])`),
with ACME and with files: every browser that can use a passkey speaks it. TLS is
rustls with aws-lc; OpenSSL makes no network connection.

**Where the keys are kept.** The ACME account key and the certificate's private
key are stored in the database, encrypted under the master key like the
server's other secrets (`server_secrets`, §9): a master key rotation
re-encrypts them, a backup carries them, a copy of the database file without the
master key does not give them. They are never written to a file.

- *Verify:* `brigid-store`'s test `acme_secrets_are_encrypted_replaced_and_survive_a_rotation`
  (no `PRIVATE KEY` in the raw row); `brigid-server`'s `tests/acme_pebble.rs`,
  run in CI against Pebble, the test authority of Let's Encrypt (a certificate
  obtained, served under its name, reloaded from the database, renewed with the
  same account).
- *Implementation reference:* [`brigid-server/src/acme.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-server/src/acme.rs) (`Certificates`, `server_config`, `Acme`); the choice of mode in `brigid-server/src/lib.rs` (`tls_mode`).
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
- **Built:** registered clients and exact redirect URIs, Authorization Code with
  PKCE (`S256`) for every client, `state` and `nonce`; `sub` = the VSID of the identity the
  person chose, `exp` five minutes, unique `jti`, EdDSA signature with stored, rotatable
  keys; a code is single use and is spent at its first presentation
  ([`protocol.md`](protocol.md) §6).

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
- **Mitigation:** `MASTER_KEY` is never stored in the database or the configuration; it is
  loaded from an env var, a mounted secret, a root-only key file, or a key sealed by the
  machine (§9).
- **Mitigation:** HKDF-SHA3-256 used to derive per-context sub-keys from the master key.

### 4.6 Cross-site scripting (XSS)
- **Mitigation:** `Content-Security-Policy` header restricts `default-src`,
  `frame-ancestors`, `object-src`, and `base-uri`. Two policies exist, by
  design:
  - API responses (JSON or a redirect, never a page) get the policy of
    `brigid-api/src/router.rs` (`build_router()`): `default-src 'none'`, with
    `frame-ancestors`, `base-uri` and `form-action` at `'none'`. Nothing can
    be loaded, framed or submitted from one of them, and no origin is named.
  - UI responses (`/`, `/signup/`, static assets, …) get the policy of
    [`server-seedling/src/ui.rs`](https://github.com/brig-id/server-seedling/blob/main/src/ui.rs)'s
    `apply_ui()`. It is applied after the interface is attached as the
    router's fallback, because in axum 0.8 a `.layer()` only wraps the routes
    that exist when it is called: the layer inside `build_router()` cannot
    reach a fallback added later.
- **Inline scripts and styles by hash:** the Qwik static (SSG) build emits
  inline `<script>`/`<style>` content (its resumability bootstrap), and with
  no per-request rendering step a nonce can't reach it. Instead `app`'s build
  writes `dist/csp-hashes.json`, the `'sha256-…'` source of every inline block,
  and `server-seedling` puts exactly those in `script-src`/`style-src` of UI
  responses ([`brig-id/app#3`](https://github.com/brig-id/app/issues/3),
  [`brig-id/core#20`](https://github.com/brig-id/core/issues/20)). Only
  well-formed hash sources are copied from the manifest. The hashes change from
  one build to the next, so the manifest ships inside each `dist/`.
- **No other origin:** the policy of UI responses names no host, scheme or
  wildcard. `default-src`, `script-src`, `style-src`, `img-src` (with
  `data:`), `font-src`, `connect-src` and `form-action` are `'self'`: the
  interface loads nothing from anywhere else ([`interface.md`](interface.md)
  §1), and a build of it that tried to would be stopped here.
- **No weaker policy:** a build of the interface without its manifest is not
  served at all. A release build of the server fails without it, and a
  directory without it is refused at start. There is no `'unsafe-inline'` in
  any policy the server can emit (until 0.4.1 a build without manifest was
  served under `'unsafe-inline'`, with a warning).
- *Verify:* `server-seedling` unit tests
  `hashes_make_the_policy_and_nothing_else_does`,
  `the_policy_names_no_origin_and_allows_no_inline_code` and
  `without_a_list_of_hashes_there_is_no_policy` (`src/ui.rs`), and
  `a_directory_that_is_not_a_build_is_refused` (`tests/static_files.rs`); in `app`,
  `scripts/check-origins.mjs` on the built files and the end-to-end test
  "the pages load nothing from another origin, and nothing is refused".

### 4.7 Cross-origin requests (CORS)
- **The interface is served from the server's own origin**, with the session in an
  `HttpOnly` cookie: the account's routes (`/auth/*`, `/account/*`, `/admin/*`) send no
  `Access-Control-Allow-Origin` at all, and refuse a change that does not come from that
  origin (§4.4). There is no setting to open them to another origin.
- **What pages that sign in need answers any origin, without credentials:** the token
  endpoint, the keys and discovery (`Access-Control-Allow-Origin: *`, no cookie).
- *Verify:* `brigid-api`'s test `an_account_route_answers_no_other_origin`.

---

## 5. Authentication Policy

### A passkey is the only way to authenticate an account

**Status:** built.

No password, one-time code, magic link, SMS, social or federated sign-in for the account,
and no "trust this device" that skips the passkey. The recovery code is not an
authentication method: it enrols one new passkey and opens nothing by itself.

A passkey is a discoverable WebAuthn credential: Windows Hello, Touch ID and Face ID, an
Android screen lock, a security key with a PIN or a fingerprint reader, a password manager.

### User verification is required, everywhere

**Status:** built. *Verify:* `brigid-api`'s test
`a_passkey_that_does_not_verify_its_user_is_refused` (a registration built by hand, with
and without the user-verified flag).

`userVerification: "required"` in every ceremony (sign-up, sign-in with or without passkey
autofill, adding a passkey, enrolling after recovery), and the server refuses a response
whose authenticator data lacks the UP or the UV flag. Each sign-in is therefore two factors
in one gesture: the device that holds the key, and what unlocks it. No second step is added
on top.

*Reference:* `brigid-webauthn/src/service.rs` uses the passkey API of `webauthn-rs`
(`start_passkey_registration`, `start_passkey_authentication`,
`start_discoverable_authentication`), which sets `UserVerificationPolicy::Required` and
checks the flag.

**"Verified" does not mean "biometric".** The authenticator reports one bit, UV: the user
was verified by a method this authenticator supports. The method is not in the signed data.
The `uvm` extension that would name it is not implemented in practice and would be the
authenticator's own statement. Every platform falls back to a PIN or passcode when the
biometric fails, and a relying party cannot forbid it. The biometric does not make the
credential stronger for the server: both unlock the same private key, locally. The
interface says "your fingerprint, your face or your screen lock", never "biometrics".

### Attestation is not requested

**Status:** built (`attestation: "none"`).

- Requiring attestation would exclude synced passkeys (iCloud Keychain, Google Password
  Manager, most password managers), which come without an attestation statement: there is
  no single device to attest. They are what most people have.
- Attestation discloses the make and model of the authenticator, which a server whose
  purpose is to learn as little as possible has no use for.
- Without it the AAGUID is an unverified hint. It is used only to suggest a name for the
  passkey, never for a decision.

What is given up: the server cannot enforce "certified hardware only".

### Passkeys must support PRF

**Status:** decided — enforced with end-to-end encryption (§7), after its measurements.

A passkey that does not support the WebAuthn PRF extension is refused, everywhere, from
sign-up: one level of protection, with no weaker mode to explain, test or audit.

---

## 6. Security Invariants

These invariants are enforced in code and must be verified by auditors:

1. **VSID stability:** same `(identity, client_id)` → same VSID (the VSID secret is a single deployment-wide value, stored encrypted, not a per-call input); different `client_id` → different VSID.
   - *Reference:* [`brigid-identity/src/vsid.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-identity/src/vsid.rs) — `identity_vsid()`
   - *Verify:* tests `deterministic`, `different_client_different_vsid`, `known_answer` in the same file.
2. **VSID isolation:** the VSID is derived from an identity's UUID only, never from an account, a username, a display name or a DID; two identities of one account have different VSIDs at the same relying party.
   - *Verify:* test `different_identity_different_vsid`; `identity_vsid()` takes a `Uuid`, so no other input can be passed.
3. **Zero raw secrets in storage:** every *sensitive* field written by `brigid-store` (credential bodies, signing-key seeds, user-supplied identifiers, etc.) MUST be encrypted with AES-256-GCM via `brigid-crypto` before the `INSERT` reaches SQLite. Non-secret *operational metadata* — table primary keys, foreign keys, `expires_at` timestamps, schema-migration markers, and the session table's token **hashes** (SHA3-256 of a 256-bit random token: a hash is not a usable session) — may be stored in plaintext so the database can be queried and pruned. What MUST never appear in plaintext is any value that would reveal a credential, a private key, a user-supplied identifier, or the contents of an active session.
   - *Reference:* [`brigid-store/src/store.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-store/src/store.rs) — see the `encrypt_field` / `decrypt_field` helpers and their call sites in every public store function.
   - *Verify:* `dump_contains_no_plaintext` integration test in `brigid-store` — it asserts the secret payloads do not appear in a raw `.dump`, but tolerates plaintext metadata columns by construction.
4. **No secrets in logs:** `tracing` spans must not capture key material, WebAuthn private keys, or OIDC signing keys.
5. **No token as an account credential:** brig·id's API authenticates the account by its
   server-side session cookie only;
   it accepts no ID token, so there is nothing to replay against it. ID tokens, once the
   relying-party flow exists, are for relying parties, single audience, five-minute lifetime.
   - *Reference:* [`brigid-api/src/session.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-api/src/session.rs) — `AccountSession`, `require_same_origin`.
6. **Key zeroization:** `OidcSigningKey` inner `SigningKey` (ed25519-dalek) implements `ZeroizeOnDrop`.

---

### Decided, not built yet

These hold by decision and are not enforced by the code described here. Each joins the list
above, with its reference and its test, when it is implemented.

- **Labels never leave:** an identity's name and icon are never sent to a relying party
  ([`model.md`](model.md) §2). *Built: the tests of the token assert its whole content, and
  the demonstration site's end-to-end suite reads what an application receives.*
- **No link across servers** (§8).
- **What describes a person is unreadable to the operator**, and only passkeys that make
  this possible are accepted (§7).
- **No way in for an administrator:** no administrator can give access back to an account,
  and a recovery code is mandatory ([`model.md`](model.md) §6–§7). *Built: no response of
  the administration holds anything of an identity, and the tests of the server assert it;
  every change needs a passkey ceremony a few minutes old at most.*
- **The secrets of links are in no log:** a setup link, an administrator's link and an
  invitation carry their secret in the fragment of the address, which is never sent
  ([`model.md`](model.md) §7); the server stores only their hash.
- **The administration log holds no name:** accounts are identifiers in it, it is
  encrypted, and its entries are deleted after twelve months by default
  ([`model.md`](model.md) §7).

---

## 7. End-to-End Encryption

**Status:** decided as an objective of v1. Designed at step 2, on a prototype and on
measurements; built at step 3, before the audit.

Storage encryption (§4.5) defeats a stolen database. It hides nothing from whoever runs the
server, who holds the master key. A product that says "you do not have to trust us" should
not ask its users to trust their server's operator with their names.

### Scope: everything that describes

Encrypted in the browser, readable by the person only: the **labels** of identities (name,
icon, description) and their **attributes** ([`model.md`](model.md) §2–§3).

The server keeps in a form it can read only what it must know to work: identifiers, which
identity belongs to which account, which identity an account presents to which relying
party, passkeys, sessions.

### Keys

- The WebAuthn **PRF** extension makes an authenticator compute a secret from a passkey and
  a salt, at sign-in. The server never sees it.
- One random **data key** per account encrypts the fields above. It is stored once per
  passkey, wrapped by a key derived from that passkey's PRF output, and once more wrapped by
  the recovery code. Adding a passkey requires a session that holds the data key.
- Synced passkeys carry their PRF secret with them.

### Only passkeys that can do it are accepted

A passkey without PRF is refused, everywhere (§5). By a third-party matrix dated September
2026 this excludes Windows Hello before the Windows 11 update of February 2026 (Windows 10
altogether) and some password managers; it keeps iCloud Keychain (iOS 18, macOS 15 and
later), Google Password Manager, recent Windows 11, security keys with `hmac-secret`,
1Password, Proton Pass. The prototype of step 2 measures this on real devices before the
rule is enforced; if the exclusion proves much wider, the rule is reconsidered first.

*Decided — step 2, not yet published:* the measurement is a test page,
`lab.brig.id` (repository `brig-id/lab`). It creates a passkey on the device in hand, asks
for a PRF output at creation and again at sign-in, and records which authenticator answered
and what it returned: an output, none, or an error. It keeps no output, only whether one
came. The project's author runs it on their own devices and the results are written here as
a table, with the device, the system, the browser and the authenticator.

### Recovery

The recovery code also decrypts. It is split in the browser into a part that proves the
right to recover, sent to the server, and a part that unwraps the data key, never sent
([`protocol.md`](protocol.md) §7). The code is mandatory, and no administrator can give
access back: someone with neither a passkey nor the code has lost the account, and nothing
can decrypt what described its identities. The interface says so at sign-up.

### Sharing with a relying party

An attribute the person agrees to share must reach the relying party. The browser decrypts
it on the linking page and hands it to the server for that sign-in; the server puts it in
the token and does not keep it.

### What it protects against, and what it does not

| Protects against | Does not protect against |
| --- | --- |
| A copy of the database, with or without the master key | An operator who changes the page the server serves, to capture the key |
| A curious administrator reading stored data | The server knowing which identities exist, whose they are, and where and when they sign in: it issues the tokens |
| A request to hand over stored data | What a relying party was given: it holds it |

The interface is served by the operator. End-to-end encryption in a web page raises the cost
of spying from "read a table" to "ship a modified page", which leaves traces. It is not a
defence against an operator determined to cheat, and brig·id does not say that it is.

### To settle with the prototype

Whether PRF is evaluated at creation or needs a first sign-in, per authenticator; the
derivations and the format of a wrapped key; how existing accounts move; what the server can
still validate (lengths, not contents); pictures as encrypted blobs; how a person is told,
before creating a passkey, that their device cannot be used.

---

## 8. Across Servers

**Status:** decided; an invariant, with nothing to build.

**A server holds no reference to an account or an identity of another server.**

An idea was examined: an employer's server aggregating the identities a person holds on
their personal server, to keep work identities unusable outside the company.

- That purpose is already met without any link: an identity exists on one server, only that
  server signs tokens for it, and only the relying parties that server accepts receive them
  ([`protocol.md`](protocol.md) §6, the operator's policy).
- "Everything is encrypted, so the employer sees nothing" does not hold. To present an
  identity a server must know that it exists, which account uses it, where and when. The
  link itself is the leak: whatever is shared into an employee's account on the employer's
  server is tied to that employee.
- Relaying (the employer's server as a relying party of the personal one) lets the employer
  see every use of the personal identity and impersonate it wherever the employer is
  trusted: worse than doing nothing.
- The direction was reversed: the server that should know the least became the place where
  everything is gathered. If anything aggregates, it is the person's own browser
  ([`model.md`](model.md) §8).

### What a relying party can trust of a server

**Status:** decided — step 2 (the rule of relying parties, written in the integration
guides); the lists below: decided — after v1.

A relying party that accepts clients describing themselves ([`protocol.md`](protocol.md) §6)
accepts people from servers it has never seen. Nothing it can check proves that a server
runs unmodified brig·id, is not compromised, or is not a server made to look like one. A
server can claim any edition and version. What the relying party verifies is narrower, and
it is enough for the main threat:

- **A token was signed by the key that the `iss` publishes over TLS.** Its `iss` is the
  server the person named, its `aud` the relying party's own `client_id`, its signature
  checks against that server's JWKS.
- **A server speaks only for its own people.** The relying party identifies a person by
  (`iss`, `sub`). A false or compromised server signs tokens with its own `iss`, so it can
  sign in only as accounts that belong to it: never as a person of another server, at any
  relying party that follows the rule.

What remains, and is not solved by brig·id:

| A server that is… | Can | Same as |
|---|---|---|
| False (made by anyone) | Create as many people as it wants | Sign-up with any e-mail address: a relying party that needs real people needs its own checks |
| Compromised | Sign in as any of its own people | A compromised mail provider. Limited by passkeys (the server never holds a person's private key, but it can sign a token without one) and by the end-to-end encryption of what describes them (§7) |
| Run by a hostile operator | The same | Choosing one's server is choosing whom to trust; the project says so to people |

**Lists published by brig.id.** *Decided — after v1.* Two signed lists, in the format of
[OpenID Federation 1.0](https://openid.net/specs/openid-federation-1_0.html) (final,
February 2026): brig.id acts as a trust anchor, publishes its entity configuration at
`/.well-known/openid-federation`, and issues **trust marks**, short-lived signed statements
about an entity:

- **About servers**, for relying parties: "this server's operator is known to the project
  and runs a released version". It states who answers for the server, not that the server is
  honest; it can be withdrawn at an incident. A relying party chooses: any server, servers
  with the mark, or its own list.
- **About clients**, for servers: clients reported for phishing or abuse. A server's
  operator chooses to follow it, follow another list, or none (`protocol.md` §6, the
  policies `open` and `listed`). A list is read periodically and applied; it is never a
  condition for a server to work.

Following a list is always the operator's or the relying party's choice, never a default
that a server cannot leave: the project must not become the place every server depends on.
Each list must say why an entry is there, and how to dispute it.

**Fetching client documents** is the new attack surface of this section: the server makes
requests to addresses chosen by anyone. The rules (special-use ranges refused on the address
actually connected to, no redirects, size and time limits, bounded cache) are in
[`protocol.md`](protocol.md) §6. A client's self-declared name is not shown in place of its
host: otherwise any site could present itself as another one.

---

## 9. Key Material Lifecycle

```
Startup: BRIGID_MASTER_KEY (env var)  OR  BRIGID_MASTER_KEY_FILE (Docker/Swarm/K8s secret file)
         ↓
         loader validates 64-char ASCII hex, decodes to 32 bytes
         ↓
         HKDF derives per-context keys (per-account storage keys, the keys
         that encrypt the stored OIDC signing seeds, the VSID secret and the
         ACME account and certificate keys)
         ↓
         OIDC signing keys and the VSID secret are read from the database
         and decrypted
         ↓
Runtime: keys held in process memory only
         ↓
Shutdown: keys dropped (zeroized on drop via secrecy/zeroize)
```

### Where the key is kept

| Kept | Read by | A copy of the database or a backup | The disk taken away | Root on the running machine |
| --- | --- | --- | --- | --- |
| A container secret (`BRIGID_MASTER_KEY_FILE`) | The platform, then the process | Protected | Depends on the platform | Not protected |
| A root-only key file (seedling's default) | systemd hands it to the service (`LoadCredential=`) | Protected | **Not protected** | Not protected |
| A key file on another disk | The same | Protected | Protected while that disk stays | Not protected |
| Sealed by the machine (`seedling init --seal`, `seal-key`) | systemd unseals it with the TPM or the host key (`LoadCredentialEncrypted=`); a command typed as root uses `systemd-creds decrypt` | Protected | **Protected** | Not protected |

A sealed key is useless off its machine, so the key is shown once when it is made: an offline
copy is the only way back after the machine or its TPM is replaced (`seal-key --from`). No
choice protects against someone who is root on the running machine, who can read the
process's memory.

### MASTER_KEY requirements
- 32 cryptographically random bytes (256 bits)
- Encoded as 64 hex characters in the `BRIGID_MASTER_KEY` env var, or stored as a Docker secret file referenced by `BRIGID_MASTER_KEY_FILE`
- Must never appear in logs, config files, error messages, or core dumps

---

## 10. Post-Quantum Readiness

See [`pqc.md`](pqc.md) for the full PQC strategy and migration plan.

Summary: `brigid-crypto` provides the X-Wing hybrid KEM (ML-KEM-768 + X25519) and composite
ML-DSA-65 + Ed25519 signatures, both following their IETF drafts and checked against the
drafts' test vectors. **Nothing in brig·id uses them yet**: OIDC tokens are signed with
Ed25519, and no data is encrypted to a KEM key. They are ready for when a use case needs them.

---

## 11. Out of Scope

- Host OS security (hardening the server running brig·id)
- Browser/client security beyond CSP and HTTPS
- Certificate authority / PKI management (operator responsibility)
- Network-level DDoS mitigation (operator responsibility — put behind a reverse proxy/CDN)
