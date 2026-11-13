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

**Hybrid combination: X-Wing** ([draft-connolly-cfrg-xwing-kem](https://datatracker.ietf.org/doc/draft-connolly-cfrg-xwing-kem/),
RustCrypto `x-wing`): `ss = SHA3-256(ss_ML-KEM || ss_X25519 || ct_X25519 || pk_X25519 || label)`.
Binding the X25519 ciphertext and public key is what keeps the hybrid IND-CCA secure when
only one component holds; the earlier `HKDF(ss_mlkem || ss_x25519)` did not (audit finding
S4). The decapsulation key is a single 32-byte seed. A non-contributory X25519 share is
refused at decapsulation. *Verify:* `crypto/tests/xwing_kat.rs` (the draft's vectors).

### 2.2 Signatures — ML-DSA-65 + Ed25519

| Parameter | Value |
|---|---|
| Standard | FIPS 204 (NIST, August 2024) |
| Classical component | Ed25519 (RFC 8032) |
| PQC component | ML-DSA-65 (Dilithium-65) |
| Security level | NIST Level 3 (ML-DSA-65 — strength comparable to SHA-384 / AES-192 collision resistance per FIPS 204) |
| Public key size | 1952 bytes (ML-DSA-65) + 32 bytes (Ed25519) |
| Signature size | 3309 bytes (ML-DSA-65) + 64 bytes (Ed25519) |

**Hybrid combination: composite ML-DSA** ([draft-ietf-lamps-pq-composite-sigs-19](https://datatracker.ietf.org/doc/draft-ietf-lamps-pq-composite-sigs/),
id-MLDSA65-Ed25519-SHA512). Both components sign the message representative

```
M' = "CompositeAlgorithmSignatures2025" || "COMPSIG-MLDSA65-Ed25519-SHA512"
     || len(ctx) || ctx || SHA-512(M)
```

ML-DSA-65 with the label as its FIPS 204 context and in hedged (randomized) mode, Ed25519 as
usual. The prefix and label stop either half from being used as a signature of `M` on its own
(no stripping), and `ctx` (≤ 255 bytes) binds a signature to its application use. The
signature is valid only if both halves are. Serialisations: signature `mldsa || ed25519`
(3373 bytes), public key (1984), private key = the two 32-byte seeds (64). *Verify:*
`crypto/tests/composite_kat.rs` (the draft's vector, from another implementation).

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

---

## 5. Migration Roadmap

### Phase A — Available now (brigid-crypto)
- [x] X-Wing (ML-KEM-768 + X25519) hybrid KEM implemented and tested against the draft's vectors
- [x] Composite ML-DSA-65 + Ed25519 signatures implemented and tested against the draft's vector
- [x] Fuzz targets for both primitives

### Phase B — Wire PQC to OIDC signing (decided — step 3 of v1)
- [ ] A composite ML-DSA-65 + Ed25519 signing key, used for the relying parties that ask for it.
      Ed25519 stays the default and there is still no RSA key. The JOSE identifier and key
      format follow the IETF draft current at implementation, and may change with it; almost
      no relying-party library can verify such a token today, which is accepted. The
      demonstration site verifies one, so that it is tested end to end
- [ ] X-Wing stays unused in v1; whether it has a role in end-to-end encryption is a question
      for that design, not a commitment
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
