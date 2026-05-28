# Operations — brig·id

> **Audience:** external security auditors and advanced contributors
> (operators running brig·id in production are the primary consumers of
> sections 1–3).
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

> **Note:** The following is illustrative pseudoconfig only. Adapt to your deployment
> environment; do not use verbatim in production without review.
>
> The snippet below uses **Docker Swarm** secrets (`external: true`, populated
> with `docker secret create`). Plain `docker compose` deployments not running
> in Swarm mode must instead use **file-based secrets** (`file: ./secrets/master_key`
> with the key material in a file the container reads via
> `BRIGID_MASTER_KEY_FILE`), or supply the env var from a `.env` file with
> filesystem permissions restricted to the orchestrator account.

```yaml
secrets:
  master_key:
    external: true   # created with: openssl rand -hex 32 | docker secret create master_key -

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
>
> The steps below MUST be followed in order. The new key material is
> generated **first** (step 3) and staged on the orchestrator host at
> `/run/secrets/master_key.new` with `0600` permissions. The database
> re-encryption in step 4 reads from that same staged file, and the Docker
> secret in step 5 is created **from that same staged file** — not a fresh
> `openssl rand` invocation. As long as `/run/secrets/master_key.new`
> survives between steps 3 and 5 (it lives on the orchestrator host's tmpfs,
> not in the container), the rotated database is guaranteed to be wrapped
> under the key the next leaf launch will receive. The runbook deletes the
> staged file with `shred -u` only after `docker secret create` has copied
> it into Swarm's encrypted raft store.

1. **Stop** the brig·id server (`SIGTERM`; wait for graceful shutdown).
2. **Back up** the SQLite database file.
3. **Generate the new MASTER_KEY** and stage it where the rotation utility
   can read it (file path or stdin — never as a shell argument):
   ```bash
   # 64-character ASCII hex = 32 bytes after decoding.
   NEW_KEY_HEX="$(openssl rand -hex 32)"

   # Stage on disk with restrictive permissions for the rotate-key step.
   install -m 600 /dev/null /run/secrets/master_key.new
   printf '%s' "$NEW_KEY_HEX" > /run/secrets/master_key.new
   ```
4. **Re-encrypt the database** with the rotation utility (to be implemented —
   see Phase 8 roadmap). The CLI **must not** accept key material as
   command-line arguments — `argv` is visible to other users via `ps`, leaks
   into shell history, crash reports, and audit logs. Pass keys as file paths
   (Docker secret mount points), via dedicated environment variables read once
   and zeroized, or via stdin:
   ```bash
   # File-based (preferred when keys live in Docker/Compose secrets):
   leaf rotate-key \
     --old-key-file /run/secrets/master_key \
     --new-key-file /run/secrets/master_key.new \
     --db /data/brigid.db

   # Stdin-based (interactive operator):
   printf '%s\n%s\n' "$OLD_KEY_HEX" "$NEW_KEY_HEX" | \
     leaf rotate-key --keys-from-stdin --db /data/brigid.db
   ```
5. **Publish the new key to the orchestrator** so the service restarts with the
   matching key material.

   The key material below is the same `NEW_KEY_HEX` staged in step 3 —
   re-read it from `/run/secrets/master_key.new`. Generating a new value here
   would break the rotation: the database has just been re-encrypted under the
   step-3 key, so launching `leaf` with anything else makes the database
   unreadable.

   Docker Swarm does **not** allow `docker secret rm` on a secret that is still
   referenced by a service, even if its tasks are stopped. The supported
   procedure is therefore a **two-name rotation**: create a new secret under a
   new name, update the service to use it, then delete the old secret.

   ```bash
   # Reuse the key material staged in step 3 — NOT a freshly generated value.
   # `BRIGID_MASTER_KEY_FILE` reads the file as text and tolerates a single
   # trailing newline; the file must NOT contain raw binary or any other
   # encoding.

   # 1. Create the new secret under a new name (e.g. suffixed with a date or
   #    monotonic version). `docker secret create` reads the key material
   #    directly from the staged file path — it never appears on argv or in
   #    shell history.
   docker secret create master_key_v2 /run/secrets/master_key.new
   shred -u /run/secrets/master_key.new   # wipe the staging copy

   # 2. Update the service to mount the new secret in place of the old one.
   #    `--secret-rm` detaches `master_key`; `--secret-add` attaches
   #    `master_key_v2` at the same in-container target path so
   #    `BRIGID_MASTER_KEY_FILE` does not need to change. This forces a
   #    rolling restart of the leaf tasks.
   docker service update \
     --secret-rm master_key \
     --secret-add source=master_key_v2,target=master_key \
     leaf

   # 3. Only now can the old secret be removed — it is no longer referenced.
   docker secret rm master_key

   # 4. (Optional) Rename for the next rotation: create master_key from
   #    master_key_v2's value so the next rotation reuses the same names.
   ```

   Stack-file equivalent: bump the `secrets:` entry name in `compose.yaml`,
   then `docker stack deploy -c compose.yaml brigid` — Swarm performs the
   same swap atomically per task.
6. **Restart** the server.
7. Verify with `curl https://example.com/health`.
8. **Destroy** the old key material securely.

---

## 2. OIDC Signing Key Rotation

The OIDC signing key (`OidcSigningKey`) is generated in memory at startup from the
MASTER_KEY-derived entropy. Rotating the MASTER_KEY implicitly rotates the signing key.
*Implementation:* [`brigid-crypto` — `hkdf::derive_user_key`](https://github.com/brig-id/crypto/blob/350892951381f57e34d3ce9f983915860431c063/src/hkdf.rs); called at server startup in [`server-leaf/src/main.rs`](https://github.com/brig-id/server-leaf/blob/3c87d7d660ad61a2a228515ff831c48450da47ba/src/main.rs).

For zero-downtime rotation:
1. Add the new signing key to the JWKS endpoint (dual-key mode — planned Phase B).
2. Wait for in-flight tokens to expire.
3. Remove the old signing key from JWKS.

---

## 3. Incident Response

### Suspected MASTER_KEY compromise

1. **Immediately** stop all brig·id instances.
2. Rotate the MASTER_KEY (see above) — this limits future exposure but does **not** restore
   confidentiality for data encrypted before the compromise; all data encrypted under the
   compromised key must be treated as exposed. Follow step 3 to re-encrypt.
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
