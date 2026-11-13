# brig·id — specification

brig·id is a self-hosted sign-in server. A person signs in with a passkey and nothing else.
Their account is private; an application meets an identity the person chose for it, and
each application receives a different identifier.

This repository is the single description of what brig·id is and how it works, written for
security auditors and for anyone who wants to check the project's claims. It contains no
code.

## How to read it

Each document covers one subject and says, section by section, where things stand:

| Status | Meaning |
| --- | --- |
| **Built** | In the code on `main` of the repositories. The section references the implementation and its tests |
| **In review** | Written, in an open pull request |
| **Decided — step N** | Decided, to be built in that step of v1. Nothing in the code yet, or only in part |
| **Decided — after v1** | Decided, and deliberately not part of v1 |
| **Open** | Not decided. Said so, with what is known |

A sentence in a section marked *Decided* is a commitment about the design, not a statement
about a running server.

## Documents

| Document | Subject |
| --- | --- |
| [`model.md`](model.md) | Accounts, identities, what an application receives, the life of an account, administration, servers |
| [`protocol.md`](protocol.md) | Cryptographic parameters, passkey ceremonies, sessions, the sign-in of applications, tokens, the recovery code |
| [`security-model.md`](security-model.md) | Threats, trust boundaries, attack surfaces, invariants, end-to-end encryption |
| [`pqc.md`](pqc.md) | Post-quantum algorithms and their use |
| [`interface.md`](interface.md) | The web interface: self-contained, design system, icons, typography, structure |
| [`editions.md`](editions.md) | What v1 is, the three editions, the licence, what is paid |
| [`operations.md`](operations.md) | Keys, rotation, incidents, deployment, releases |
| [`audit-checklist.md`](audit-checklist.md) | What an auditor is asked to check |
| [`audit/`](audit/README.md) | How the audit is to be funded, and the drafts of the requests |

The order of work is not here: it is tracked as issues, in the
[roadmap](https://github.com/brig-id/roots/issues/25).

## Words

| Word | Meaning |
| --- | --- |
| **Account** | A person's private way in: passkeys, a recovery code, nothing that describes them. No application ever sees it |
| **Identity** | What a person presents to the outside. An account has several |
| **Label** | The name, icon and description of an identity: for the person only |
| **Attribute** | Information about the person carried by an identity, shared with an application only after acceptance |
| **Application** | What an identity signs in to through brig·id: a website, a mobile or desktop application. The protocol documents say *relying party* |
| **Used by** / **Connected to** | The two directions of an identity's links: what uses it (applications, sessions), and what it uses (services, providers) |
| **VSID** | Virtual Stable Identifier: the identifier an application receives, one per identity and per application |
| **Server** | One installation of brig·id, on one domain, in one of three editions: seedling, grove, forest |

## Earlier references

Decisions used to be kept as numbered records ("ADR-005 §2"). Their content now lives in
the documents above, and the records were removed; the history of this repository keeps
them. Code, commits and issues written before still cite the numbers:

| Was | Subject | Now in |
| --- | --- | --- |
| ADR-001 | A note on a token-validation library | Removed with the validation code |
| ADR-002 | Accounts and identities | [`model.md`](model.md) §1–§2 |
| ADR-003 | Usernames, passkey-only sign-up | [`model.md`](model.md) §1, §6 |
| ADR-004 | Account recovery | [`model.md`](model.md) §6; [`protocol.md`](protocol.md) §7 |
| ADR-005 | Account sessions, tokens for applications | [`protocol.md`](protocol.md) §5–§6 |
| ADR-006 | The VSID secret is stored | [`protocol.md`](protocol.md) §2 |
| ADR-007 | Identifiers | [`protocol.md`](protocol.md) §2; [`model.md`](model.md) §2 |
| ADR-008 | The sign-in of applications; no conformance work in v1 | [`protocol.md`](protocol.md) §6 |
| ADR-009 | Passkeys only, user verification, no attestation | [`security-model.md`](security-model.md) §5 |
| ADR-010 | Funding the audit | [`audit/README.md`](audit/README.md) |
| ADR-011 | Releases and updates | [`operations.md`](operations.md) §6 |
| ADR-012 | The interface is self-contained | [`interface.md`](interface.md) §1–§2 |
| ADR-013 | The smallest edition is one executable | [`editions.md`](editions.md) §3 |
| ADR-014 | Three products, v1 in three steps | [`editions.md`](editions.md) §1–§2 |
| ADR-015 | Icons | [`interface.md`](interface.md) §4 |
| ADR-016 | Design system | [`interface.md`](interface.md) §3 |
| ADR-017 | Typography | [`interface.md`](interface.md) §5 |
| ADR-018 | Parts of the interface, administration | [`interface.md`](interface.md) §6; [`model.md`](model.md) §7 |
| ADR-019 | Editions | [`editions.md`](editions.md) §2–§6 |
| ADR-020 | Licence and what is paid | [`editions.md`](editions.md) §7 |
| ADR-021 | Identities across servers | [`model.md`](model.md) §8; [`security-model.md`](security-model.md) §8 |
| ADR-022 | What an application receives | [`model.md`](model.md) §3–§5 |
| ADR-023 | End-to-end encryption | [`security-model.md`](security-model.md) §7 |
| ADR-024 | The life of an account | [`model.md`](model.md) §6 |
| ADR-025 | What a server publishes; post-quantum signatures | [`protocol.md`](protocol.md) §6, §8; [`pqc.md`](pqc.md) |
