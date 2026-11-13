# ADR-005 — Account sessions and relying-party tokens are separate things

**Status:** Accepted

## Context

One object plays two roles today. `/auth/login/finish` returns an OIDC ID token, and that
token is also the bearer credential of the account's session on brig·id's own API
(`/auth/*`, `/account/*`). Consequences:

- `client_id` is any string the caller sends. No client is registered, there is no
  redirect, no `nonce`, no `state`: a third-party relying party (RP) has no defined way to
  obtain a token, and a token minted for an RP could be replayed at that RP until it
  expires.
- The API does not check the audience of the bearer token: any token the server signed
  opens the account endpoints, provided the caller also knows the account's `user_id`.
  That id travels in query strings, so in access logs.
- The session token is kept in `localStorage`, readable by any script on the page, and
  `/auth/session/refresh` renews it without limit.
- [ADR-002](002-accounts-and-identities.md) says a token for an RP is issued for an
  identity, never for the account. `/auth/login/finish` issues one for the account DID with
  whatever `client_id` was sent.
- The discovery document advertises `response_types_supported: ["id_token"]` without an
  `authorization_endpoint`, which no OIDC client can use.

ADR-002 left "account sessions vs RP tokens" open. This ADR closes it.

## Decision

### 1. The account session is a server-side session in a cookie

- Signing in with a passkey opens a session: 256 random bits, sent as the cookie
  `__Host-brigid_session` with `HttpOnly; Secure; SameSite=Strict; Path=/`. The server
  stores only a hash of it, with the account id and two deadlines (table `sessions`).
- **Idle timeout** one hour, extended by use. **Absolute lifetime** twelve hours from the
  passkey ceremony, never extended: after that, the user signs in again.
- Every account endpoint takes the account **from the session**. No endpoint accepts a
  `user_id` parameter.
- State-changing requests must be same-origin: `SameSite=Strict`, plus a check of the
  `Origin` header against the server's own origin. CORS is not needed by brig·id's own UI.
- Signing out deletes the session row. The JTI blacklist, which exists to revoke bearer
  session tokens, is removed with them.
- brig·id's API never accepts an ID token as a credential.

### 2. A relying party gets a token through the Authorization Code flow

> [ADR-008](008-oidc-conformance.md) §3 amends this section so that the flow can pass the
> OpenID Foundation conformance suite: RS256 next to EdDSA, confidential clients with a
> secret, PKCE mandatory for public clients and the default for confidential ones, a
> UserInfo endpoint, and the standard request parameters.

- RPs are **registered**: `client_id`, display name, exact `redirect_uris`. A request
  naming an unknown client or an unregistered redirect is refused before any user
  interaction.
- `GET /authorize` (Authorization Code, PKCE with `S256` mandatory, `state` and `nonce`
  honoured) runs under the account session. The user picks the **identity** to present to
  that RP; the choice is remembered per (account, client) and can be changed.
- `POST /token` exchanges the single-use, short-lived code (and the PKCE verifier) for an
  ID token:
  - `iss`: the server; `aud`: the `client_id`; `sub`: the VSID of the chosen identity for
    that client; `nonce`; `auth_time`; `iat`; `exp` five minutes after `iat`; `jti`.
  - No `server` claim (it repeats `iss`) and no `alias_type`.
- The discovery document is published with `authorization_endpoint`, `token_endpoint`,
  `response_types_supported: ["code"]`, `code_challenge_methods_supported: ["S256"]`.

### 3. Until the flow of §2 exists

No endpoint issues a token for a caller-supplied `client_id`:
`POST /account/identities/{id}/token` is removed, `/auth/login/finish` opens a session
and returns no token, and `/.well-known/openid-configuration` is not served. The JWKS and
the signing-key rotation stay, ready for §2.

## Consequences

- `protocol.md` §4–§6 and §8, `security-model.md` §4.3–§4.4b and the audit checklist are
  rewritten with the implementation.
- `core`: session store and cookie extractor replace `AuthenticatedClaims`; the six copies
  of "recompute the VSID for `user_id`" disappear; `brigid-oidc` keeps issuance and loses
  bearer validation and the JTI store.
- `app`: no token or account id in `localStorage`; requests rely on the cookie.
- `server-leaf`: `security.session_ttl_seconds` becomes the idle timeout it claims to be,
  next to a new absolute lifetime.
- Existing sessions end at the upgrade. No relying party depends on today's tokens.
- An XSS can still act within the page while it is open, but can no longer take the
  session elsewhere or keep it alive past twelve hours.

## References

- [ADR-002](002-accounts-and-identities.md)
- OpenID Connect Core 1.0 §3.1; RFC 7636 (PKCE); RFC 6265bis (`__Host-` cookies, `SameSite`)
