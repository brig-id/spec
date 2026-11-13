# ADR-008 — OIDC conformance: the OpenID Foundation suite in CI for v1, certification after

**Status:** Accepted (2026-11-13). The points marked **Amendment** change
[ADR-005](005-account-sessions-and-rp-tokens.md) §2 and wait for the owner's confirmation.

## Context

[ADR-005](005-account-sessions-and-rp-tokens.md) §2 (Authorization Code flow for relying
parties) is accepted and not implemented: brig·id issues no token today. When it does, "it
speaks OpenID Connect" must be something a third party can check, not a sentence on a website.

The OpenID Foundation publishes the conformance suite it uses for certification. It is open
source (MIT), runs locally in containers, and contains the same test plans as the hosted
instance. Certification itself is a paid, declarative process run on the hosted instance.

## Decision

### 1. The conformance suite runs in CI (v1)

- From the day ADR-005 §2 exists, CI starts a local instance of the suite and runs it against
  a server built from the commit under test, with two plans:
  `oidcc-basic-certification-test-plan` (Basic OP) and
  `oidcc-config-certification-test-plan` (Config OP).
- The job passes on the certification rule: no test `FAILED` or `INTERRUPTED`; `PASSED`,
  `WARNING`, `REVIEW` and `SKIPPED` are accepted. Every accepted warning is listed in a
  checked-in expectations file with its reason, so a new warning fails the job.
- **v1 does not ship without both plans green on `main`.** The public wording is then "passes
  the OpenID Foundation conformance suite (Basic OP, Config OP)", never "certified".
- Each authorization in the suite needs a passkey ceremony. The CI runner drives a real
  browser with a virtual authenticator through the URLs the suite exposes. The server gets
  **no test-only way around the passkey**: a bypass that exists can be enabled.

### 2. Official certification comes after v1

Certification is on the roadmap after v1, for the deployment that then exists. Conditions,
checked on 2026-11-13:

| | |
| --- | --- |
| Fee, OpenID Connect | USD 700 (members) or USD 3,500 (non-members) per new deployment; one payment covers as many profiles as wanted within a calendar year; OP and RP are separate deployments |
| Open-source waiver | Possible, case by case: the requester is an OIDF member in good standing and connected to the deployment, and asserts that no primary maintainer is paid for work on the project; one waiver per project per year, all profiles submitted at once; request to `certification@oidf.org` |
| Membership | Required for the waiver and the lower fee. Individual dues were USD 50 at the last published increase (2021); to confirm at enrolment |
| Process | Self-certification: run the plans on the hosted suite against a publicly reachable deployment, publish the logs, sign the Certification of Conformance |
| What it says | That a named deployment passed the tests of the listed profiles. It says nothing about security |

### 3. What the suite requires of ADR-005 §2

Read from the suite's source (`OIDCCBasicTestPlan`, `OIDCCCheckDiscEndpointIdTokenSigningAlgValuesSupported`):

- **Amendment — RS256.** The discovery check fails with "RS256 support is required" when
  `id_token_signing_alg_values_supported` lacks `RS256` (OpenID Connect Core §15.1 makes it
  mandatory). brig·id signs with Ed25519 only. The server therefore holds two signing keys,
  RSA (3072 bits) and Ed25519, both stored and rotated like today's key. A client is
  registered with its `id_token_signed_response_alg`: `RS256` by default (what every OIDC
  library verifies), `EdDSA` on request. The defences against algorithm confusion stay: one
  algorithm per `kid`, no `none`, no HMAC. RSA signing uses `aws-lc-rs`, already in the
  dependency tree through rustls, not the `rsa` crate (RUSTSEC-2023-0071).
- **Amendment — client authentication.** The Basic OP plan uses confidential clients:
  `client_secret_basic` (two clients) and `client_secret_post` (one). Registered clients are
  therefore of two kinds: confidential (a server-generated 256-bit secret, shown once, stored
  hashed) and public (no secret, PKCE).
- **Amendment — PKCE.** ADR-005 §2 says `S256` is mandatory. Most modules of the plan send no
  `code_challenge`; an OP that refuses them cannot pass. PKCE `S256` stays mandatory for
  public clients and the default for confidential ones; a confidential client can be
  registered with `require_pkce = false`, in which case the client secret and `nonce` protect
  the code as in OpenID Connect Core. `plain` is never accepted. To confirm on the first run
  that the plan cannot be run with PKCE on every request.
- **UserInfo endpoint.** The plan tests it (GET, POST with header, POST with body). `/token`
  therefore also returns an opaque access token, five minutes, valid at `/userinfo` only.
  `/userinfo` returns the same `sub` as the ID token.
- **Scopes and claims.** `openid` and `profile`. `profile` releases the identity's display
  `name` and `updated_at`, nothing else; an identity has no email, phone or address, by
  design, and those scopes are answered without claims (expected result: warnings). The
  identity picker shows what the relying party will receive.
- **Request parameters.** `prompt=login` (new passkey ceremony), `prompt=none`
  (`login_required` or `interaction_required` when it cannot answer silently), `max_age` with
  the `auth_time` claim, `id_token_hint`, `login_hint`, `display`, `ui_locales`,
  `claims_locales`, `acr_values` accepted; unknown parameters ignored; a code-flow request
  without `nonce` succeeds; `request` and `request_uri` answered `request_not_supported` /
  `request_uri_not_supported`; authorization requests accepted by POST too.
- **Codes.** Single use; a second use fails and revokes the tokens issued from the first.
  `redirect_uri` compared exactly. No refresh token is issued (test skipped).

## Consequences

- `core`: the points of §3 join the implementation of ADR-005 §2; `protocol.md` §1, §5 and §6
  and `pqc.md` §4 are updated with the code (RS256 next to EdDSA, UserInfo, discovery).
- `server-leaf`: the conformance job, its runner and the expectations file.
- The site may state the suite result once the job is green on `main`, with that wording.
- Certification is tracked as a post-v1 item; the fee or the waiver request is the owner's
  decision at that time.

## References

- Conformance suite: <https://gitlab.com/openid/conformance-suite> (MIT, `LICENSE.txt`)
- OP testing instructions: <https://openid.net/certification/connect_op_testing/>
- Fee schedule: <https://openid.net/certification/fees/>
- Open-source waiver: <https://openid.net/certification/open-source-project-certification-policy/>
- OpenID Connect Core 1.0 §15.1 (mandatory to implement for OPs)
- [ADR-005](005-account-sessions-and-rp-tokens.md), [ADR-007](007-identifiers.md)
