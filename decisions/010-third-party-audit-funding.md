# ADR-010 — Third-party security audit: NLnet first, OSTIF second, public call later

**Status:** Accepted (2026-11-13)

## Context

brig·id has had internal reviews only. The documents in this repository are written for an
external auditor ([`audit-checklist.md`](../audit-checklist.md)), and
[spec#3](https://github.com/brig-id/spec/issues/3) plans a public call for audit. An audit of
an identity provider costs tens of thousands of euros; the project has no revenue.

The intention was to apply to NLnet's NGI Zero Commons Fund and to OSTIF. Checked on
2026-11-13:

- **NGI Zero Commons Fund is closed.** Its thirteenth and final call closed on 2026-06-01.
- NLnet's open calls are now **Restack** and **CodeSupply** (both funded by the European
  Commission under the Open Internet Stack initiative), plus a general open call. One
  deadline for all: **2027-01-03, 12:00 CET**; then the 3rd of every odd month.
- **OSTIF** takes requests through an intake form, at any time.

## Decision

### 1. NLnet, Restack fund, call closing 2027-01-03

- **Restack** is the fund to select: "securing the core infrastructure and foundational
  technologies", including "full stack security" and "alternatives to closed online
  technologies". Grants of EUR 5,000 to 50,000 for a first proposal; results under a
  recognised free or open-source licence. CodeSupply (software supply-chain metadata) is not
  our subject.
- **What NLnet brings for the audit.** NLnet's support services for funded projects include
  a security audit by Radically Open Security and an accessibility audit; the Restack page
  promises "accessibility scans and security audits". Which services apply "can differ from
  programme to programme": **ask NLnet to confirm that the security audit applies to Restack
  before relying on it.** The proposal asks for it explicitly.
- The grant funds the work that leads to an auditable v1 (ADR-005 §2, conformance in CI,
  the self-contained app, the single executable); the audit comes at the end of it.
- Draft: [`audit/applications/nlnet-restack.md`](../audit/applications/nlnet-restack.md).

### 2. OSTIF

- OSTIF scopes the audit with the project, writes the request for proposals, selects the
  auditors, manages the engagement and publishes the report. It helps find sponsors; it does
  not fund from its own budget as a rule. It gives USD 30,000 to 200,000 as the range of a
  first audit.
- brig·id today is pre-1.0 and has no deployment beyond its own. It is a weak candidate for a
  sponsored audit. The intake is still worth sending now: the scoping conversation and the
  preparation list are useful before v1, and the request can be completed after it.
- Draft: [`audit/applications/ostif.md`](../audit/applications/ostif.md).

### 3. Public call (spec#3)

Kept for later, after v1 and after the funded paths have answered.

### 4. Rules for the applications

- **The owner sends them, in their own words.** NLnet requires proposals written by a human and
  a declaration of any use of generative AI (model, prompts, outputs), and asks whether the
  existing code contains AI-generated code. The drafts in this repository were produced with
  an AI model: they are working material to rewrite, and that use is declared.
- **The licence must stay free.** NLnet and OSTIF fund free and open-source software only.
  [ADR-020](020-licensing-and-pricing.md) must not move any audited component to a
  source-available licence while a grant is sought or running.
- **Nothing is promised that is not built.** Each draft separates what exists, what is
  decided and what is planned, like the public site.

### 5. Audit scope and readiness

Scope: `crypto` (encryption at rest, key derivation, hybrid primitives), `core` (WebAuthn
ceremonies, sessions, the relying-party flow, the encrypted store), and the shipped server.
An audit is requested only once ADR-005 §2 is implemented, the conformance suite runs in CI
([ADR-008](008-oidc-conformance.md)) and features are frozen for v1.

## Consequences

- Before 2027-01-03: the owner rewrites and submits the NLnet proposal (a token must first be
  requested by email from NLnet, as its form explains).
- The OSTIF intake can be sent at any time.
- spec#3 stays open, marked as after v1.

## References

- NLnet funds and deadlines: <https://nlnet.nl/funding.html>, <https://nlnet.nl/restack/>,
  <https://nlnet.nl/codesupply/>, <https://nlnet.nl/commonsfund/> (closed)
- NLnet proposal form: <https://nlnet.nl/propose/>; support services:
  <https://nlnet.nl/NGI0/services/>
- OSTIF: <https://ostif.org/get-an-audit/>
