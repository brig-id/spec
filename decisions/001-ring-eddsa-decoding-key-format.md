# ADR-001 — `jsonwebtoken` + ring: EdDSA DecodingKey expects raw 32-byte public key

**Status:** Accepted

## Context

`brigid-oidc` uses `jsonwebtoken v9` for JWT issuance and validation with the EdDSA
algorithm (Ed25519). The crate `ed25519-dalek v2` is used for key generation.

When wiring up token validation, all roundtrip tests failed with `InvalidSignature`
despite the token being correctly signed.

## Investigation

`jsonwebtoken v9` delegates EdDSA operations to `ring` internally.
`ring::signature::UnparsedPublicKey::new(&ED25519, key_bytes)` expects the public key
as **raw 32 bytes** (the Ed25519 public key point on the curve).

`ed25519-dalek`'s `to_public_key_der()` (from the `pkcs8` feature) returns a
**SubjectPublicKeyInfo DER** structure (44 bytes), which is the standard encoding for
public keys in PKCS#8 and X.509. `ring` does not parse this format for Ed25519.

The `jsonwebtoken` API `DecodingKey::from_ed_der(bytes)` is misleadingly named — it
does NOT accept SubjectPublicKeyInfo DER for Ed25519 verification; it passes the bytes
directly to ring's `UnparsedPublicKey`, which expects raw 32 bytes.

By contrast, `EncodingKey::from_ed_der(bytes)` correctly accepts PKCS#8 DER (48 bytes)
for the private key.

## Decision

In [`brigid-oidc/src/key.rs`](https://github.com/brig-id/core/blob/645f8dbe2223e43fdce39bfaf00868f630c4e47f/crates/brigid-oidc/src/key.rs),
`OidcSigningKey::decoding_key()` is implemented as:

```rust
pub(crate) fn decoding_key(&self) -> jsonwebtoken::DecodingKey {
    // ring expects raw 32-byte public key bytes, NOT SubjectPublicKeyInfo DER
    jsonwebtoken::DecodingKey::from_ed_der(self.signing_key.verifying_key().as_bytes())
}
```

`VerifyingKey::as_bytes()` returns the raw 32-byte point, which is what ring expects.

For the encoding (signing) key:

```rust
pub(crate) fn encoding_key(&self) -> jsonwebtoken::EncodingKey {
    let der = self.signing_key.to_pkcs8_der()
        .expect("Ed25519 PKCS8 DER encoding never fails");
    jsonwebtoken::EncodingKey::from_ed_der(der.as_bytes())
}
```

`to_pkcs8_der()` (via `pkcs8::EncodePrivateKey` trait) returns 48-byte PKCS#8 DER,
which `ring` correctly parses for EdDSA signing.

## Consequences

- Correct: roundtrip `issue_token` → `validate_token` passes.
- Fragile: the `from_ed_der` name is misleading. If `jsonwebtoken` ever changes this
  behavior (e.g., to accept DER for public keys), the raw-bytes approach would break.
  Pin `jsonwebtoken` to a minor version and test on upgrade.
- The JWKS endpoint uses `base64ct::Base64UrlUnpadded::encode_string(vk.as_bytes())`
  for the `x` parameter, consistently using raw 32 bytes.

## References

- [ring source — Ed25519 public key format](https://github.com/briansmith/ring/blob/main/src/ec/curve25519/ed25519/signing.rs)
  (raw 32-byte little-endian compressed public key, consistent with RFC 8032)
- [jsonwebtoken v9 — eddsa.rs](https://github.com/Keats/jsonwebtoken/blob/master/src/crypto/eddsa.rs)
- [ed25519-dalek — VerifyingKey::as_bytes()](https://docs.rs/ed25519-dalek/latest/ed25519_dalek/struct.VerifyingKey.html)
