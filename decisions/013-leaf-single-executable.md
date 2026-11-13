# ADR-013 — leaf is a single executable

**Status:** Accepted (2026-11-13)

## Context

Running `leaf` today takes a container runtime, a compose file, a reverse proxy (Caddy) for
TLS, and a directory of interface files next to the binary. That is a lot for the smallest
edition: a personal server, one service among others on a small machine.

OpenBao is the model: one binary, which serves its own interface and, since its 2.2 series,
obtains its own certificates over ACME.

## Decision

### 1. One file

`leaf` is one executable per target (Linux x86-64 and arm64 first). It contains:

- the web interface, embedded at compile time, compressed, served with content-hash caching;
- the database engine (SQLite, already compiled into the binary);
- TLS (rustls) and an ACME client.

It needs no container runtime, no reverse proxy, no Node.js and no database server. A data
directory and a master key are all it has outside itself.

The container image remains, as another packaging of the same binary. A reverse proxy in
front remains possible. Neither is required any more.

### 2. Built-in TLS, three modes

| Mode | For |
| --- | --- |
| `acme` | The default for a public name: certificates obtained and renewed from an RFC 8555 authority (Let's Encrypt by default, any directory URL otherwise, so a private authority works). TLS-ALPN-01 on port 443, so port 80 is optional. The ACME account key and the certificate keys are stored in the data directory, encrypted like the other secrets. |
| `files` | A certificate and key provided by the operator (today's `tls_cert` / `tls_key`). |
| `off` | Plain HTTP behind a reverse proxy that terminates TLS (today's reference deployment). |

The process never runs as root. Binding ports 443 and 80 uses `CAP_NET_BIND_SERVICE` or
socket activation; a systemd unit is shipped.

To settle with the implementation: the ACME crate (`rustls-acme` or `instant-acme`, both pure
Rust), and whether the built-in listener accepts TLS 1.3 only (every browser that has
passkeys has TLS 1.3).

### 3. First start

`leaf init` creates the data directory and a configuration with the server's name, and
prints what to do next. For the master key, two documented choices:

- a key file created by `leaf init` (owner-only permissions): simple, and it protects copies
  of the database (backups, a dump), **not** the theft of the whole disk;
- the key supplied by the environment or a service credential, kept off the data disk: what
  the security model assumes today.

The first administrator is created through a one-time setup link printed at first start
([ADR-018](018-app-structure.md)).

### 4. A static binary

The binary links to the system's OpenSSL today, only because `webauthn-rs` depends on it to
verify attestation certificates, which brig·id never asks for
([ADR-009](009-passkeys-only-user-verification.md)). A single file that runs anywhere must
not need the host's libraries: the release build is a static musl binary with OpenSSL
vendored. Removing OpenSSL altogether (upstream change, or another WebAuthn library) would
be a security-relevant change with its own ADR.

### 5. What does not change

- No self-update: a server that replaces itself from the network is an attack surface.
  Updating is replacing the file and restarting.
- A release build without the interface, or without its CSP hash manifest, fails. The
  fallback to `'unsafe-inline'` when the manifest is missing
  ([`security-model.md`](../security-model.md) §4.6) disappears with the directory on disk.
- `ui_dist_dir` remains as a development option to serve interface files from disk.

## Consequences

- `server-leaf`: embedding, the ACME mode, `leaf init`, static release binaries published
  with checksums, SBOM and build provenance, next to the image.
- `app`: its build output is consumed at compile time; [ADR-012](012-self-contained-app.md)
  keeps it small (target: interface and images under 2 MB).
- `security-model.md` §4.1 (transport) and `operations.md` are rewritten with the
  implementation: the built-in listener becomes the reference, the proxy the variant.
- `helm.brig.id` keeps running the image.

## References

- OpenBao, ACME TLS listeners: <https://openbao.org/docs/rfcs/acme-tls-listeners>
- RFC 8555 (ACME), RFC 8737 (TLS-ALPN-01)
- `rustls-acme`, `instant-acme`, `rust-embed` (crates.io)
- [ADR-009](009-passkeys-only-user-verification.md), [ADR-012](012-self-contained-app.md),
  [ADR-014](014-three-products.md)
