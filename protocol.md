# Protocol — brig·id

> **Audience:** external security auditors and advanced contributors.
> Cryptographic parameters, passkey ceremonies, sessions, the sign-in of relying parties,
> tokens and the recovery code. Statuses are explained in the [README](README.md).

**Words.** This document says *relying party* for what the interface and the other
documents call an *application*: a website or an application that people sign in to through
brig·id. An *identity* belongs to an *account*; a relying party never sees the account.

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

**Status:** built.

VSID (Virtual Stable Identifier) is the stable, opaque `sub` claim of ID tokens. It gives one
identity a **different identifier at each relying party**, and two identities of one account
different identifiers at the same relying party: neither relying parties among themselves
nor one relying party alone can link them.

### Formula

The keying material is the **VSID secret**: one value per deployment, the same
for every identity. It is 32 random bytes generated when the database is
created and stored in it, encrypted with a key derived from `MASTER_KEY`
(the secret is stored, not derived from the master key at each start, so that
rotating the master key changes no VSID).

```
VSID = HKDF-SHA3-256(
    ikm  = vsid_secret,                   // deployment-wide, stored encrypted
    salt = none,
    info = "brigid-vsid-v2" ||
           identity_uuid (16 bytes) ||
           u32_be(len(client_id)) || client_id
)[..32]
→ encoded as unpadded base64url (43 chars)
```

The identity's UUID has a fixed length and `client_id` is length-prefixed: the input is
unambiguous. `v2` keeps it apart from the earlier construction, whose input was a DID.

### Invariants

1. Same `(identity, client_id)` → same VSID (deterministic; the VSID secret is constant per
   deployment, not a per-call input).
2. Different `client_id` → different VSID (unlinkable across relying parties).
3. Different identity → different VSID at the same relying party, including two identities
   of one account.
4. The input is an identity's UUID, never an account, a username, a display name or a DID.
   Renaming anything, moving the server to another domain or rotating `MASTER_KEY` changes
   no VSID (see the rotation runbook in [`operations.md`](./operations.md)).

*Reference implementation:* [`brigid-identity/src/vsid.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-identity/src/vsid.rs) — `identity_vsid()`, tests `deterministic`, `different_client_different_vsid`, `different_identity_different_vsid`, `client_id_is_length_prefixed`, `known_answer`;
[`brigid-store/src/store.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-store/src/store.rs) — `vsid_secret()`

---

## 3. Passkey Registration

**Status:** built.

```
Client                          brigid-api                      brigid-store
  │                                 │                               │
  │─── POST /auth/register/begin ──▶│                               │
  │    { }                           │                               │
  │                                 │ generate a username           │
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
  │◀── 204, Set-Cookie (session) ───│                               │
```

Sign-up asks for nothing but the passkey: the server generates the account's username,
which the person never types ([`model.md`](model.md) §1),
and the account exists only once the passkey is stored. Registration opens the session.

Whether an account may be created is the sign-up policy ([`model.md`](model.md) §7).
`register/begin` takes an optional `{ "link": "…" }`: an invitation, or the setup link of
a server without account. The link is checked at `begin` and **spent at `finish`, in the
transaction that creates the account**: a link that reached its number of accounts between
the two creates none. A refusal is `403` with a `code`: `signup_closed`,
`invitation_required`, or `link_invalid` (unknown, spent, expired or revoked, which the
server does not tell apart).

### Security checks (registration)
- RP ID must exactly match the server domain (no wildcard).
- Challenge must be single-use (deleted after verification).
- The server part of the username must be this server's domain: an account cannot be
  registered under a domain the server does not control.
- Attestation is not requested (`attestation: "none"`, the `webauthn-rs` passkey
  profile): the server learns no authenticator make or model, and does not restrict
  which authenticators may register.

---

## 4. Passkey Authentication

**Status:** built.

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

Signing in opens an **account session** (§5); it issues no token. brig·id's own API never
takes an ID token as a credential.
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
- User verification is required in every ceremony, and a response without the UV flag is
  refused ([`security-model.md`](security-model.md) §5).

---

## 5. Account Session

**Status:** built.

The session of an account with this server is server-side, behind a random token in a
cookie. It is not a token for anyone else, and no endpoint accepts a `user_id` parameter:
the account always comes from the session.

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
| `GET /auth/session` | `200 { idle_expires_at, expires_at, recovery_code, admin, confirmed_at }`, or `401` when there is no live session, or its account is gone or suspended. `recovery_code`: whether the account has one. `admin`: whether it administers the server. `confirmed_at`: when the latest passkey ceremony of this session was done (§10) |
| `POST /auth/logout` | deletes the session row and clears the cookie (`204`). The token is unknown from then on, across restarts too |

*Reference:* [`brigid-api/src/session.rs`](https://github.com/brig-id/core/blob/main/crates/brigid-api/src/session.rs),
`EncryptedStore::{create_session, use_session, delete_session}`.

---

## 6. The Sign-in of Relying Parties

**Status:** built, for everything in this section not marked otherwise. *Verify:* the
end-to-end suite of the demonstration site ([`brig-id/demo`](https://github.com/brig-id/demo)),
which signs in through a real server with an ordinary OpenID Connect library, as a
confidential and as a public client.

A relying party obtains an ID token for an **identity**, never for the account, through the
OpenID Connect Authorization Code flow.

### Registered clients

A relying party is registered on the server: a `client_id`, a name shown to people, and the
exact `redirect_uris` it may use (`https`, or `http` on the loopback host for development).

| Kind | Authentication at the token endpoint |
|---|---|
| Confidential | A secret of 256 random bits generated by the server, shown once, stored as its SHA3-256 hash. `client_secret_basic` or `client_secret_post` |
| Public | None: PKCE alone ties the client to the code |

A registration also lists the **attributes** the relying party may ask for, each one
required or optional ([`model.md`](model.md) §4); none by default.

### `GET /authorize` (also `POST`)

1. The client and its redirect URI are checked first, by exact comparison. If either is
   wrong, the browser is sent to this server's own page with an error, and nothing is sent
   anywhere else.
2. Any other error goes back to the relying party as an OAuth error, with its `state` and
   the `iss` parameter (RFC 9207): `response_type` other than `code`, a scope without
   `openid`, a missing or malformed PKCE challenge, `request` or `request_uri` (not
   supported), `prompt=none` combined with another value; and the
   `profile` or `email` scope from a relying party registered for none of that scope's
   attributes is `invalid_scope`; it asked for more than it may. A scope the server does
   not know is ignored.
3. **PKCE with `S256` is mandatory for every client**, confidential ones included. `plain` is
   never accepted.
4. The navigation comes from another site, so it carries no session cookie
   (`SameSite=Strict`). The request is held (ten minutes, in memory, with a cap) and the
   browser is sent, by a `303`, to the interface page `/continue/?request=<id>`. From there
   every call is same-origin.

`profile` and `email` ask for the attributes of that scope the
relying party is registered for, no others. There is no `claims` parameter.

`state` and `nonce` are returned unchanged. `prompt=login` requires a passkey ceremony more
recent than the request; `max_age` compares with the time of the last ceremony;
`prompt=none` is answered without any page; unknown parameters are ignored.

### The person's answer

Same-origin endpoints, called by the linking page ([`model.md`](model.md) §5):

| Endpoint | Effect |
|---|---|
| `GET /auth/authorize/{request}` | The relying party's registered name; whether a passkey ceremony is needed; the account's identities, with their attributes, and the one last presented to this relying party; the attributes asked for, each required or optional; per identity, what was already decided about them |
| `POST …/approve { identity_id, attributes }` | Issues a code for that identity and remembers the choice for this relying party. `attributes` names what the person accepts, of what was asked; the rest of it is refused. A required attribute refused is `409` and the request stays. Requires the session, and an identity of the session's account |
| `POST …/deny` | Returns the relying party's address with `access_denied` |
| `POST …/silent` | For `prompt=none`: a code if a session is live, an identity was already chosen and every attribute asked for is settled; else `login_required`, `interaction_required` or `consent_required` |

What a code will give is fixed when it is issued: the attributes asked for by the request,
accepted by the person, with the value the identity had at that moment. It is read back
from what was recorded, never from the body of the answer.

A request is answered once. The answer is the address to send the browser to: the redirect
URI with `code`, `state` and `iss`.

### `POST /token`

- The client authenticates (table above). An unknown client and a wrong secret get the same
  answer, `invalid_client`.
- The **code** is single use, valid one minute, and is taken out at its first presentation,
  whatever follows. It is refused unless the client, the `redirect_uri` and the PKCE
  `code_verifier` are those of the request that obtained it; and if the identity was deleted
  in the meantime.
- The response carries the ID token, and an `access_token` because OAuth 2.0 requires the
  field: a random value that no endpoint accepts.

### ID token

```json
{ "alg": "EdDSA", "typ": "JWT", "kid": "<RFC 7638 thumbprint>" }
```

```json
{
  "sub":       "<vsid>",        // §2: this identity, at this relying party
  "iss":       "https://example.com",
  "aud":       "<client_id>",
  "nonce":     "<from the request, if any>",
  "auth_time": 1234567890,      // the passkey ceremony behind the session
  "iat":       1234567890,
  "exp":       1234568190,      // five minutes
  "jti":       "<uuid>"
}
```

**By itself it describes neither the person nor the identity.** A claim that is the same
at two relying parties would let them link their users; a test asserts that, when no
attribute is shared, the payload holds these claims and no other.

The attributes the person accepted for this relying party are added
as the claims of their names (`given_name`, `family_name`, `preferred_username`, `email`,
`locale`, `zoneinfo`), and `email_verified: false` with `email`. **An identity's label (its
name, icon and description) has no claim**: a test shares every attribute and looks for the
label in everything sent to the relying party.

| Choice | Why |
|---|---|
| **EdDSA (Ed25519) by default, no RSA key** | One modern algorithm, one key type to protect. OpenID Connect Core §15.1 makes RS256 mandatory: a relying party whose library cannot verify EdDSA cannot use brig·id, and this is accepted |
| **No UserInfo endpoint** | The ID token is enough; a second kind of token is avoided. Added at the first real need |
| **Attributes in the ID token** | With `openid` alone a relying party receives an identifier. `profile` and `email` add attributes ([`model.md`](model.md) §3–§5) to the ID token, only for what the person accepted. No second token, no second endpoint to protect |
| **No conformance or certification claim** | The OpenID Foundation's conformance suite requires RS256, requests without PKCE and UserInfo. Running it, and certification, are not part of v1; brig·id says that it implements the Authorization Code flow, and nothing more |

*Decided — step 3:* a relying party can ask for ID tokens signed with the composite
ML-DSA-65 + Ed25519 signature ([`pqc.md`](pqc.md)).

*Reference (core#92):* `brigid-api/src/routes/authorize.rs`, `routes/token.rs`;
`brigid-oidc/src/token.rs`, `pkce.rs`; `brigid-store` (tables `clients`,
`client_identities`); integration tests `a_site_signs_a_person_in_and_learns_an_identifier_only`,
`an_authorization_request_is_checked_before_anyone_is_asked`,
`the_token_endpoint_gives_nothing_without_the_client_the_code_and_the_verifier`,
`prompt_and_max_age_decide_whether_the_person_is_asked`. *Attributes (core#91):*
`brigid-identity/src/attributes.rs`; `brigid-store` (table `grants`);
`an_application_receives_an_attribute_only_once_its_exact_value_was_accepted`,
`refusing_an_optional_attribute_continues_and_a_required_one_stops`,
`the_label_of_an_identity_appears_in_no_token`.

### What an identity gave

Same-origin endpoints of the account, for the page of one identity
([`model.md`](model.md) §5):

| Endpoint | Effect |
|---|---|
| `GET /account/identities/{id}/applications` | The relying parties the identity signed in to: name, first and latest sign-in, the attributes accepted with their values and dates, the ones refused |
| `DELETE …/applications/{client_id}/attributes/{name}` | Forgets the decision about one attribute: no longer sent, asked again |
| `DELETE …/applications/{client_id}` | Removes everything recorded for that relying party, and the remembered choice of this identity there |

---

## 7. Recovery Code

**Status:** built. The split of the code is decided — step 3.

A recovery code is generated by the server, never chosen by the person: a chosen passphrase
would have little entropy and need a slow hash.

| Bytes | Content |
|---|---|
| 8 | **Recovery id**: random; names the account through a keyed index, like usernames. It is neither the account's id nor its username |
| 16 | **Secret**: random, 128 bits |
| 1 | Check byte: a typo is reported before any request |

Shown as 40 base32 characters in groups of four.

- **Stored:** the keyed index of the recovery id (recomputed when the master key rotates),
  and a verifier of the secret, `HKDF-SHA3-256(ikm = secret, info =
  "brigid-recovery-verifier-v1")`, both encrypted with the account's key. Never the code.
  The verifier needs no key: 128 random bits cannot be guessed from it.
- **Flow:** `POST /auth/recovery/begin { code }` → a passkey registration challenge →
  `POST /auth/recovery/finish` stores the new passkey, consumes the code and opens the
  session. It enrols one passkey and does nothing else ([`model.md`](model.md) §6).
- **Guards:** the rate limit of the ceremonies, constant-time comparison of verifiers, and
  no lockout.
- **An account without a recovery code is still in its onboarding** (core#98).
  `/account/identities` and everything under it, and the answer to an authorization
  request, are refused (`403`) until `POST /account/recovery-code` has run; the linking
  page is told so (`needs_recovery_code`) and shown nothing of the account. Its session,
  its passkeys and the code itself stay open. There is no endpoint that removes a code.

*Decided — step 3:* with end-to-end encryption the code is split in the browser into a part
that proves, sent to the server, and a part that decrypts, never sent
([`security-model.md`](security-model.md) §7).

*Reference:* `brigid-identity/src/recovery.rs`; `brigid-api/src/routes/recovery.rs`.

---

## 8. What a Server Publishes

### `GET /.well-known/openid-configuration`

**Status:** built. The endpoints of §6, `response_types_supported: ["code"]`,
`subject_types_supported: ["pairwise"]`, `id_token_signing_alg_values_supported: ["EdDSA"]`,
`scopes_supported: ["openid", "profile", "email"]`,
`code_challenge_methods_supported: ["S256"]`. It, the JWKS
and the token endpoint can be read from any origin: none of them depends on a cookie.

### `GET /.well-known/jwks.json`

**Status:** built.

JWK Set containing the Ed25519 public keys that verify tokens: the current signing
key first, then any key retired by a rotation and still within its grace period
(see [`operations.md`](./operations.md) §2). A token names its key in the `kid` header, and
a relying party verifies it with the key of that name.

```json
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "x":   "<base64url of 32-byte raw public key>",
    "kid": "<RFC 7638 thumbprint>",
    "alg": "EdDSA",
    "use": "sig"
  }]
}
```

### `GET /auth/server`

**Status:** built. What differs from one server to another and the interface needs to know
before anyone is signed in: `{ "sign_up": true | false, "policy": "open" | "invitation" |
"closed" }`. `sign_up` says whether anyone can create an account without a link, `policy`
which of the three policies applies ([`model.md`](model.md) §7). Same origin only.

The server publishes no DID document: `/.well-known/did.json` repeated the key that the
JWKS publishes and nothing read it, so it was removed. A relying party finds a server
through the two documents above.

---

## 9. Deleting an Account

**Status:** built.

| Endpoint | Effect |
|---|---|
| `POST /account/delete/begin` | With the session: a passkey authentication challenge for the session's account, held as a ceremony whose purpose is deletion |
| `POST /account/delete/finish` | With the session and the passkey's answer: the account is deleted and the cookie cleared (`204`) |

- **A ceremony has one purpose.** One begun to sign in is unknown to `delete/finish`, and
  one begun to delete is unknown to `/auth/login/finish`.
- The ceremony is begun for the session's account, and its challenge names that account's
  passkeys only. `finish` also checks that the account whose passkey answered is the
  session's.
- Deleting removes the account's row, and with it its passkeys, recovery code, sessions,
  identities with their attributes, the identity remembered for each relying party and
  what was shared there. Nothing is kept: no tombstone, no log of who it was. A relying
  party is not told.

*Reference:* `brigid-api/src/routes/account.rs`, `EncryptedStore::delete_account`; tests
`deleting_an_account_takes_its_own_passkey_ceremony_and_leaves_nothing`,
`deleting_an_account_leaves_no_row_about_it`,
`an_account_without_a_recovery_code_can_do_nothing_else`.

---

## 10. Administration

**Status:** built. What it is for and what it never shows: [`model.md`](model.md) §7.

Every endpoint takes the account from the session, like the rest. **Reading needs an
administrator's session. A change also needs a passkey ceremony done in that session five
minutes ago at most** (`security.admin_confirmation_seconds`): the ceremony that opened the
session counts, and so does a later one:

| Endpoint | Effect |
|---|---|
| `POST /auth/confirm/begin` | With the session: a passkey authentication challenge for the session's account, held as a ceremony whose purpose is confirmation |
| `POST /auth/confirm/finish` | With the session and the passkey's answer: the session's `confirmed_at` is now (`204`) |

A change asked later than that is refused with `403` and the code `confirmation_required`;
nothing was done, and the same request succeeds after a confirmation. Like deletion (§9),
the ceremony has one purpose and names the session's account only.

| Endpoint | Effect |
|---|---|
| `POST /admin/claim` | `{ link }`, with any account's session: the account becomes an administrator. The link was made on the machine; `403 link_invalid` otherwise |
| `GET /admin/accounts` | The accounts: identifier, generated username, creation date, role, suspension, last sign-in, number of passkeys, whether a recovery code exists, and whether it is the session's own |
| `POST /admin/accounts/{id}/suspend`, `…/unsuspend` | Suspends the account, or lifts it |
| `PUT /admin/accounts/{id}/administrator` | `{ admin }`: grants or withdraws the role |
| `DELETE /admin/accounts/{id}` | Deletes the account, as §9 does |
| `GET /admin/applications` | The registered clients: identifier, name, return addresses, whether it has a secret, what it may ask for |
| `POST /admin/applications` | Registers a client (§6). The response holds the secret of a confidential one: the only time it is readable |
| `PUT /admin/applications/{client_id}/attributes` | Replaces what the client may ask for |
| `DELETE /admin/applications/{client_id}` | Removes the client, and what identities had shared with it |
| `GET /admin/sign-up` | The sign-up policy and the invitations that still open something: identifier, accounts left, end date. Never the link |
| `PUT /admin/sign-up` | `{ policy }` |
| `POST /admin/invitations` | `{ uses, days }` (1 to 100, 1 to 90): the response holds the link, the only time it is readable |
| `DELETE /admin/invitations/{id}` | Revokes the invitation |
| `GET /admin/log` | The latest entries, newest first: when, what, by whom (an administrator or the machine), about which account, a detail. Usernames are looked up at this moment; an account that is gone has none |

- **Links.** A setup link, an administrator's link and an invitation are 128 random bits,
  base64url. The server stores their hash, their kind, how many accounts they still open
  and their end. The address that carries one puts it in the fragment (`/setup/#link=…`,
  `/admin/claim/#link=…`, `/signup/#invitation=…`), which no request carries; the
  interface sends it in a request body.
- **A suspended account** has no session (they are deleted when it is suspended), and is
  refused at sign-in and recovery with `403` and the code `suspended`. `POST /token`
  answers `invalid_grant` for a code issued to one of its identities: a relying party
  cannot tell why.
- **The last administrator.** Withdrawing the role from, suspending or deleting the last
  administrator who is not suspended is `409`; so is suspending or deleting one's own
  account from these endpoints, and deleting one's account (§9) while being that last
  administrator.
- **The log** is one encrypted record per action, with the acting account and the account
  concerned as identifiers. Deleting an account does not rewrite it: the identifier then
  names nothing. Entries older than the retention are deleted at each start and each write.

*Reference:* `brigid-api/src/routes/admin.rs`, `brigid-api/src/session.rs`
(`Administrator`, `ConfirmedAdministrator`), `brigid-store/src/admin.rs`; tests
`a_fresh_server_takes_no_account_until_its_setup_link_is_used`,
`only_an_administrator_reaches_the_administration`,
`no_administration_response_holds_anything_of_an_identity`,
`an_administrative_change_needs_a_passkey_ceremony_done_a_moment_ago`,
`a_suspended_account_is_closed_and_keeps_everything`,
`an_invitation_opens_a_server_to_as_many_accounts_as_it_says`,
`the_role_is_claimed_with_a_link_made_on_the_machine_and_the_log_names_nobody_who_left`.
