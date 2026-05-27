# Post-Quantum Cryptography — brig·id

> **Audience:** external security auditors and advanced contributors.
> This document describes the PQC algorithm choices, hybrid construction rationale,
> and migration roadmap for brig·id.

---

## 1. Threat Context

Standard elliptic-curve cryptography (ECDH, Ed25519) is vulnerable to large-scale quantum
computers running Shor's algorithm. While such computers do not exist today, the
"harvest now, decrypt later" attack pattern means long-lived data encrypted today
could be decrypted in the future.

brig·id adopts a **hybrid classical+PQC** strategy: both classical and PQC algorithms
are used simultaneously. A break in one does not compromise the other.

---

## 2. Selected Algorithms

### 2.1 Key Encapsulation (KEM) — ML-KEM-768 + X25519

| Parameter | Value |
|---|---|
| Standard | FIPS 203 (NIST, August 2024) |
| Classical component | X25519 (Diffie-Hellman on Curve25519) |
| PQC component | ML-KEM-768 (Kyber-768) |
| Security level | ~128-bit classical (X25519); NIST Level 3 PQC (ML-KEM-768 — strength comparable to AES-192 key search) |
| Public key size | 1184 bytes (ML-KEM-768) + 32 bytes (X25519) |
| Ciphertext size | 1088 bytes (ML-KEM-768) + 32 bytes (X25519 ephemeral) |

**Hybrid combination:** shared secrets from X25519 and ML-KEM-768 are concatenated
and fed into HKDF-SHA3-256 to produce the final shared key.

*Reference:* [`src/kem.rs`](https://github.com/brig-id/crypto/blob/350892951381f57e34d3ce9f983915860431c063/src/kem.rs)

### 2.2 Signatures — ML-DSA-65 + Ed25519

| Parameter | Value |
|---|---|
| Standard | FIPS 204 (NIST, August 2024) |
| Classical component | Ed25519 (RFC 8032) |
| PQC component | ML-DSA-65 (Dilithium-65) |
| Security level | NIST Level 3 (ML-DSA-65 — strength comparable to SHA-384 / AES-192 collision resistance per FIPS 204) |
| Public key size | 1952 bytes (ML-DSA-65) + 32 bytes (Ed25519) |
| Signature size | 3309 bytes (ML-DSA-65) + 64 bytes (Ed25519) |

**Hybrid combination:** both signatures are produced independently and concatenated.
Verification requires both to be valid.

*Reference:* [`src/dsa.rs`](https://github.com/brig-id/crypto/blob/350892951381f57e34d3ce9f983915860431c063/src/dsa.rs)

---

## 3. Hybrid Construction Rationale

Pure PQC algorithms are newer and have received less cryptanalysis than classical ones.
Using hybrids ensures:

1. **Backward security:** if ML-KEM or ML-DSA have unforeseen weaknesses, X25519/Ed25519 still protect.
2. **Forward security:** if a quantum computer breaks X25519/Ed25519, ML-KEM/ML-DSA still protect.
3. **Standard compliance:** NIST explicitly recommends hybrid approaches during the transition period.

---

## 4. Current Deployment Status

| Use case | Current algorithm | PQC status |
|---|---|---|
| OIDC token signing | Ed25519 (classical) | PQC hybrid available in brigid-crypto; not yet wired to OIDC |
| Storage key derivation | HKDF-SHA3-256 (quantum-resistant) | SHA-3 is quantum-resistant (Grover's: security halved, still 128-bit) |
| WebAuthn credential storage | AES-256-GCM (quantum-resistant) | AES-256 is quantum-resistant (Grover's: security halved, still 128-bit) |
| DID key material | Ed25519 (classical) | Migration to hybrid planned |

---

## 5. Migration Roadmap

### Phase A — Available now (brigid-crypto)
- [x] ML-KEM-768 + X25519 hybrid KEM implemented and tested
- [x] ML-DSA-65 + Ed25519 hybrid signatures implemented and tested
- [x] Fuzz targets for both primitives

### Phase B — Wire PQC to OIDC signing (planned)
- [ ] `OidcSigningKey` extended to support ML-DSA-65 + Ed25519 hybrid
- [ ] JWKS endpoint extended with hybrid public key
- [ ] `id_token_signing_alg_values_supported` updated to `["EdDSA", "ML-DSA-65+Ed25519"]`

### Phase C — Key rotation and agility (planned)
- [ ] Multiple signing keys in `JwkSet` (one classical, one hybrid)
- [ ] Token validation accepts both
- [ ] Configurable preference order in `leaf.toml`

---

## 6. NIST References

- [FIPS 203 — ML-KEM](https://csrc.nist.gov/pubs/fips/203/final) (Module-Lattice-Based Key-Encapsulation Mechanism)
- [FIPS 204 — ML-DSA](https://csrc.nist.gov/pubs/fips/204/final) (Module-Lattice-Based Digital Signature Algorithm)
- [NIST IR 8413 — Status of PQC Standardization](https://csrc.nist.gov/publications/detail/nistir/8413/final)

---

*Last updated: Phase 8 (audit readiness). Covers Phases 1–7.*
