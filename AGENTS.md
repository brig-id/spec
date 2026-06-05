# AGENTS.md — brig·id `spec`

This repository contains the **public technical specification** for brig·id,
intended for third-party security auditors and the open-source community.

## Language

**All content must be in English** — documents, diagrams, checklists. No exceptions.

## Scope

- `security-model.md` — threat model, attack vectors, mitigations, security assumptions
- `protocol.md` — WebAuthn flow, OIDC flow, VSID computation, cryptographic parameters
- `pqc.md` — post-quantum algorithm choices and migration plan
- `audit-checklist.md` — checklist for third-party auditors
- `operations.md` — operational procedures (MASTER_KEY rotation, incident response)

This repository contains **no code**. It is documentation only.

## Rules

- Write for an external security auditor, not an internal developer.
- Every security claim must reference the implementation (crate, function, test).
- Never include secrets, private keys, or production configuration examples.
- All cryptographic parameters must cite their NIST/IETF standard.
- Keep documents self-contained — do not assume the reader has access to the source code.

## Key documents

| File | Content |
| --- | --- |
| `security-model.md` | Threat actors, attack surfaces, mitigations, trust assumptions |
| `protocol.md` | Step-by-step WebAuthn + OIDC flows, VSID formula, key sizes |
| `pqc.md` | ML-KEM-768 (FIPS 203), ML-DSA-65 (FIPS 204), hybrid rationale, migration roadmap |
| `audit-checklist.md` | Critical points, attack vectors to test, contact + CVD process |
| `operations.md` | MASTER_KEY rotation procedure, incident response playbook |

## Commit conventions

Format: `type(scope): <emoji> description`

| Type | Emoji | When |
| --- | --- | --- |
| `feat` | ✨ | New document or section |
| `fix` | 🐛 | Correction to an existing document |
| `docs` | 📝 | Minor wording or formatting update |
| `chore` | 🔧 | Maintenance, config |
| `ci` | 👷 | CI/CD |
| `revert` | ⏪ | Reverts a previous commit |

### Allowed scopes

| Scope | Maps to |
| --- | --- |
| `protocol` | `protocol.md` |
| `operations` | `operations.md` |
| `audit` | `audit-checklist.md` |
| `pqc` | `pqc.md` |
| `security` | `security-model.md` |
| `ci` | `.github/workflows/` |

**Do not use a scope outside this list.** If a new document is added, update this table
and `.vscode/settings.json`.

```text
feat(protocol): ✨ document VSID computation formula
fix(audit): 🐛 correct ML-KEM key size reference
docs(operations): 📝 add MASTER_KEY rotation example
```
