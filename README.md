# Application Outcome & Status Standard (AOSS)

**v0.1 — draft specification, published for comment.**

AOSS is reusable infrastructure for digital onboarding and application outcomes in U.S. financial services: a common status taxonomy, error model, and API contract for checking/savings account opening and credit application intake, plus native SDK components to build the flows themselves.

## The problem

Every institution that launches a deposit or credit product assembles roughly the same machine: a mobile intake flow, an identity/KYC integration, fraud signals, sometimes manual review, an eligibility or underwriting decision, and a way to tell the applicant what happened. Each vendor and internal system reports status in its own vocabulary, so teams rebuild — repeatedly, per product and per channel — the logic that answers three questions:

- **Where is this application?** (inconsistent, ambiguous statuses across systems and surfaces)
- **What went wrong?** (vendor-specific errors leaking into application code and, worse, into applicant messaging)
- **What happens next?** (unclear outcomes, unactionable "pending" states, adverse decisions communicated without the structured reasons downstream notices need)

The integration glue is undifferentiated, but getting it wrong is expensive: abandoned applications, support load, audit gaps, and compliance risk. AOSS standardizes the glue.

## What AOSS defines

**1. A canonical status model.** One taxonomy, used on every surface:

`started → submitted → under_review`, with `under_review ⇄ needs_more_information` as needed, resolving to `approved` or `declined` (deposit accounts continue `approved → opened`); proposed terminal extensions `expired` and `withdrawn`.

**2. A backend middle layer contract.** A normalization service between clients and decision/verification systems: standard error categories with defined retryability, outcome objects pairing machine-readable status with human-readable explanation and structured next steps, idempotent operations, correlation IDs end to end, and audit-ready event logging. Adverse outcomes carry specific, ranked reasons so adopting creditors can populate the notices their regulatory obligations require.

**3. Companion native SDKs (separate repository).** iOS and Android onboarding components that implement this standard as clients: configurable flows for account opening and credit intake, disclosure and consent capture, accessibility-ready components (WCAG 2.1 AA), and verbatim rendering of AOSS statuses, reasons, and next steps. Requirements live in [`aoss-onboarding-sdk`](https://github.com/kameroli/aoss-onboarding-sdk).

The SDKs and the middle layer are designed to work together but are adoptable independently.

## What's in this repository

| File                           | Contents                                                                                                                                                                                 |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`SPEC.md`](SPEC.md)           | The standard: status model and state machine, error taxonomy, outcome objects, audit/observability requirements, adverse-outcome communication guidance, versioning, conformance levels. |
| [`openapi.yaml`](openapi.yaml) | OpenAPI 3.0 contract for the middle layer's core API: submit application, get status, provide information, list next steps, status event history, and the status-change webhook.         |
| [`ADAPTERS.md`](ADAPTERS.md)   | The vendor adapter interface: contract, example mapping tables (identity verification; decision/underwriting), retry/idempotency rules, sandbox adapter for testing.                     |
| [`TESTING.md`](TESTING.md)     | Contract testing approach, adapter conformance checklist, and failure-mode test categories.                                                                                              |

## Project status

- **Specification:** v0.1 draft, open for comment. Expect breaking changes between 0.x minors; changes will be recorded in `CHANGELOG.md`.
- **Reference implementations:** design phase. Requirements for the iOS SDK are published in [`aoss-onboarding-sdk`](https://github.com/kameroli/aoss-onboarding-sdk); implementation of the iOS SDK and a Spring Boot middle layer follows per the roadmap below. Android follows the same specification.
- No production adoption is claimed for v0.1; the draft is published to gather implementer feedback first.

## Roadmap

1. **Specification** _(current)_ — stabilize the v0.1 taxonomy, API contract, and adapter interface through public comment.
2. **SDK foundations** — iOS SDK and Spring Boot middle-layer reference implementations, with the sandbox adapter and conformance test suite; Android SDK groundwork.
3. **Pilot integrations** — exercise the spec against real vendor sandboxes and pilot flows; fold findings back into the spec.
4. **v1.0 hardening** — freeze the core taxonomy and API under semantic versioning; publish the conformance suite as the compatibility gate.

## Feedback

Open a GitHub issue for anything: taxonomy gaps, transition-rule disagreements, error categories you need, adapter-contract friction. The v0.1 draft specifically requests comment on the `expired`/`withdrawn` extension states, the error category set, the next-step type vocabulary, and counteroffer handling. For substantial proposals, an issue describing the use case before a PR is appreciated.

## License

Apache-2.0 — see [`LICENSE`](LICENSE)..

---

Maintained by Melissa Rojas.
