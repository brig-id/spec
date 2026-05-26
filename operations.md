# Operations — brig·id

> **Audience:** operators and security auditors.
> This document describes operational procedures: key rotation, incident response,
> and deployment security.

---

## 1. MASTER_KEY Management

### Generating a new MASTER_KEY

```bash
# Generate 32 cryptographically random bytes, hex-encoded
openssl rand -hex 32
# Example output (do NOT use this): ab12cd34ef56...
```

The 64-character hex string is the value of `BRIGID_MASTER_KEY`.

### Supplying the key at runtime

**Docker Compose (recommended):**
```yaml
secrets:
  master_key:
    external: true   # created with: docker secret create master_key <(openssl rand -hex 32)

services:
  leaf:
    image: brigid/leaf
    secrets:
      - master_key
    environment:
      BRIGID_MASTER_KEY_FILE: /run/secrets/master_key
```

**Environment variable (development only):**
```bash
export BRIGID_MASTER_KEY="$(openssl rand -hex 32)"
```

**Never:** embed the key in `leaf.toml`, Docker images, or version control.

### MASTER_KEY rotation procedure

> ⚠️ Key rotation requires re-encrypting all data in the database. Plan for downtime.

1. **Stop** the brig·id server (`SIGTERM`; wait for graceful shutdown).
2. **Back up** the SQLite database file.
3. Run the key rotation utility (to be implemented — see Phase 8 roadmap):
   ```bash
   leaf rotate-key --old-key "$OLD_KEY" --new-key "$NEW_KEY" --db /data/brigid.db
   ```
4. Update the Docker secret:
   ```bash
   docker secret rm master_key
   printf "%s" "$NEW_KEY" | docker secret create master_key -
   ```
5. **Restart** the server.
6. Verify with `curl https://example.com/health`.
7. **Destroy** the old key material securely.

---

## 2. OIDC Signing Key Rotation

The OIDC signing key (`OidcSigningKey`) is generated in memory at startup from the
MASTER_KEY-derived entropy. Rotating the MASTER_KEY implicitly rotates the signing key.

For zero-downtime rotation:
1. Add the new signing key to the JWKS endpoint (dual-key mode — planned Phase B).
2. Wait for in-flight tokens to expire.
3. Remove the old signing key from JWKS.

---

## 3. Incident Response

### Suspected MASTER_KEY compromise

1. **Immediately** stop all brig·id instances.
2. Rotate the MASTER_KEY (see above) — this invalidates *forward confidentiality*
   for any data encrypted before the rotation but does **not** automatically
   re-encrypt existing ciphertext. Follow step 3 to do so.
3. Re-encrypt the database with the new key (decrypt all rows with the old key,
   re-encrypt with the new key). Until this step is complete, an attacker who
   exfiltrated old ciphertext can still decrypt it with the old key.
4. Passkey re-registration is **not** required — only public key material
   (credential IDs and attestation) is stored; the passkey private key never
   leaves the authenticator.
5. Invalidate all active OIDC sessions at relying parties.
6. Report to security contact (see `SECURITY.md`).

### Suspected WebAuthn credential compromise

1. Identify the compromised credential via the `credential_id` in logs.
2. Delete the credential from the database:
   ```sql
   -- Via the admin API (to be implemented)
   DELETE FROM credentials WHERE credential_id = '...';
   ```
3. Notify the user to register a new passkey.

### Token replay attack detected

If the `JtiStore` reports a replay:
- Log only minimal non-PII fields: `jti`, `kid`, `aud`, `iss`, timestamps, and
  a truncated HMAC of `sub` (never the raw `sub` value). **Never log the raw JWT
  string or the full claims set** — `sub` is a VSID that functions as a stable
  pseudonymous identifier and must be treated as PII.
- The token is already rejected — no further action needed for that request.
- Investigate the source IP for credential compromise.
- Consider revoking the affected `client_id` if the RP is compromised.

---

## 4. Deployment Security Checklist

### Before going to production

- [ ] `BRIGID_MASTER_KEY` is supplied via Docker secret (not env var)
- [ ] TLS certificate is valid and from a trusted CA
- [ ] `domain` in `leaf.toml` matches the TLS certificate's CN/SAN
- [ ] `cors_origins` in config contains only your RP domain(s), no `*`
- [ ] `read_only: true` set on the container filesystem in compose.yaml
- [ ] `USER nonroot:nonroot` in Dockerfile (non-root container)
- [ ] Docker image is `gcr.io/distroless/cc-debian12` (no shell)
- [ ] `cargo audit` passes with no unignored advisories
- [ ] Firewall rules: only ports 80 and 443 exposed
- [ ] Reverse proxy (nginx/Caddy) in front for DDoS mitigation

---

## 5. Monitoring and Alerting

Key events to monitor via structured logs (`tracing` JSON output):

| Event | Log field | Alert threshold |
|---|---|---|
| Rate limit triggered | `event = "rate_limit"`, `ip` | Sustained > 100/min → investigate |
| JTI replay detected | `event = "jti_replay"` | Any occurrence |
| WebAuthn assertion failed | `event = "auth_failed"` | > 10/min per IP |
| Server startup with missing key | Server refuses to start | Always alert |
| DB error | `event = "db_error"` | Any occurrence |

---

*Last updated: Phase 8 (audit readiness). Covers Phases 1–7.*
