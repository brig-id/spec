# AGENTS.md — brig·id `spec`

This repository contains the **public technical specification** for brig·id,
intended for third-party security auditors and the open-source community.

## Language

**All content must be in English** — documents, diagrams, checklists. No exceptions.

## Scope

One document per subject. Each is the single description of its subject: there are no
separate decision records. A choice is written where it applies, with its reason in a
sentence when the reason is not obvious.

| File | Content |
| --- | --- |
| `README.md` | The map, the statuses, the words, and where earlier "ADR" references now point |
| `model.md` | Accounts, identities, what an application receives, the life of an account, administration, servers |
| `protocol.md` | Cryptographic parameters, passkey ceremonies, sessions, the sign-in of relying parties, tokens, the recovery code |
| `security-model.md` | Threats, trust boundaries, attack surfaces, authentication policy, invariants, end-to-end encryption |
| `pqc.md` | Post-quantum algorithm choices and their use |
| `interface.md` | The web interface: self-contained, design system, icons, typography, structure |
| `editions.md` | What v1 is, the three editions, the licence, what is paid |
| `operations.md` | Keys, rotation, incidents, deployment, releases |
| `audit-checklist.md` | Checklist for third-party auditors |
| `audit/` | How the audit is funded; drafts of the requests |

This repository contains **no code**. It is documentation only.

## Rules

- Write for an external security auditor, not an internal developer.
- **Every section says where it stands** (`**Status:**` built, in review, decided for a
  step, decided for after v1, open), as defined in `README.md`. A section marked *built*
  references the implementation (crate, function, test). Nothing may read as built when it
  is not.
- When something is decided, change the document of its subject. Do not add a record of the
  decision next to it: the history of the repository is the record.
- Never include secrets, private keys, or production configuration examples.
- All cryptographic parameters must cite their NIST/IETF standard.
- Keep documents self-contained — do not assume the reader has access to the source code.
- The order of work is not here: it lives in the organisation's GitHub Project.

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
| `model` | `model.md` |
| `protocol` | `protocol.md` |
| `security` | `security-model.md` |
| `pqc` | `pqc.md` |
| `interface` | `interface.md` |
| `editions` | `editions.md` |
| `operations` | `operations.md` |
| `audit` | `audit-checklist.md`, `audit/` |
| `docs` | `README.md`, `AGENTS.md`, changes across several documents |
| `ci` | `.github/workflows/` |

**Do not use a scope outside this list.** If a new document is added, update this table
and `.vscode/settings.json`.

```text
feat(protocol): ✨ document VSID computation formula
fix(audit): 🐛 correct ML-KEM key size reference
docs(operations): 📝 add MASTER_KEY rotation example
```
