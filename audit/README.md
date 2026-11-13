# Funding the audit — brig·id

> How a third-party security audit is to be paid for, and the drafts of the requests.
> What an auditor is asked to check is in [`audit-checklist.md`](../audit-checklist.md).
> Statuses are explained in the [README](../README.md).

---

## 1. The plan

**Status:** decided. Nothing is sent yet.

brig·id has had internal reviews only. An audit of an identity provider costs tens of
thousands of euros, and the project has no revenue.

1. **NLnet, first.** Checked on 2026-11-13: the NGI Zero Commons Fund is closed (its last
   call closed on 2026-06-01). NLnet's open calls are Restack and CodeSupply, with one
   deadline: **2027-01-03, 12:00 CET**, then the 3rd of every odd month.
   - **Restack** is the fund to select: "securing the core infrastructure and foundational
     technologies", including "full stack security" and "alternatives to closed online
     technologies". EUR 5,000 to 50,000 for a first proposal; results under a recognised free
     licence.
   - NLnet's support services for funded projects include a security audit and an
     accessibility audit; which services apply "can differ from programme to programme".
     **To confirm with NLnet for Restack before relying on it.**
   - The grant funds the work that leads to an auditable v1; the audit comes at the end.
   - Draft: [`applications/nlnet-restack.md`](applications/nlnet-restack.md).
2. **OSTIF, second.** It scopes an audit with the project, selects the auditors, manages the
   engagement and publishes the report; it helps find sponsors. It gives USD 30,000 to
   200,000 as the range of a first audit. A pre-1.0 project with one deployment is a weak
   candidate for a sponsored audit; the first contact is still worth making for the
   preparation it brings.
   - Draft: [`applications/ostif.md`](applications/ostif.md).
3. **A public call for audit**, after v1 and after the funded paths have answered.

## 2. Rules for the requests

- **The project's author sends them, in their own words.** NLnet requires proposals written
  by a human and a declaration of any use of generative AI, and asks whether the existing
  code contains AI-generated code. The drafts here were produced with an AI model: they are
  working material to rewrite, and that use is declared.
- **The licence stays free.** NLnet and OSTIF fund free software only
  ([`editions.md`](../editions.md) §7).
- **Nothing is promised that is not built.** Each draft separates what exists, what is
  decided and what is planned.

## 3. Scope and readiness

Scope: the cryptography library (encryption at rest, key derivation, hybrid primitives), the
core (passkey ceremonies, sessions, the sign-in of applications, the encrypted store,
end-to-end encryption), and the shipped servers.

An audit is requested at step 3 of v1 ([`editions.md`](../editions.md) §1): features frozen,
and these documents re-read against the code.

## References

- NLnet: <https://nlnet.nl/funding.html>, <https://nlnet.nl/restack/>,
  <https://nlnet.nl/propose/>, <https://nlnet.nl/NGI0/services/>
- OSTIF: <https://ostif.org/get-an-audit/>
