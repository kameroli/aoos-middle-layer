# Changelog

All notable changes to the Application Outcome & Status Standard (AOSS) are recorded here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versioning follows the policy in `SPEC.md` Section 8 (semantic versioning; breaking changes may occur between 0.x minor versions while the standard is in draft).

Entries distinguish **normative** changes (conformance impact: statuses, transitions, error categories, required fields, API operations) from **editorial** changes (clarifications with no conformance impact).

## [Unreleased]

### Changed (editorial)

- SPEC.md: added an explicit division-of-responsibility statement — institutions decide and own outcome substance; the middle layer standardizes structures and enforces transparency guarantees; clients render with fidelity.
- SPEC.md: documented authentication, authorization, and session management as out of scope; transport credentials are issued and validated by adopter infrastructure, and adopters enforce caller-scoped visibility of application resources.
- SPEC.md: stated that `started` is client-local in v0.1; the middle layer's record begins at submission, and transitions from `started` other than `application.submitted` are reserved for a future version with an intake-registration operation (addresses #1).
- README: corrected project status (reference implementations are in design phase), fixed the license statement to match the committed Apache-2.0 LICENSE, reframed the native SDKs as a companion implementation in a separate repository, and corrected the status-model summary to show terminal branching rather than a linear chain.

### Fixed

- ADAPTERS.md: "stateless (or externally stated)" corrected to "stateless (or with externally stored state)".
- TESTING.md: closed an unterminated italic in the footer.

## [0.1.0] - 2026-09-02

### Added

- Initial public draft of the standard, published for comment:
  - `SPEC.md` — canonical status taxonomy and state machine, error taxonomy, outcome object model, audit and observability requirements, adverse-outcome communication guidance, versioning policy, conformance levels.
  - `openapi.yaml` — middle-layer API contract v0.1.0: `submitApplication`, `getApplication`, `provideInformation`, `listNextSteps`, `listStatusEvents`, and the `statusChanged` webhook.
  - `ADAPTERS.md` — vendor adapter interface: contract and boundary rules, example mapping tables (identity verification; decision/underwriting), retry and idempotency rules, sandbox adapter requirements.
  - `TESTING.md` — contract testing approach, state-machine verification, adapter conformance checklist, failure-mode test categories, outcome quality checks.
- Proposed extension statuses `expired` and `withdrawn`, published specifically for adopter comment.

[Unreleased]: https://github.com/kameroli/aoss-middle-layer/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/kameroli/aoss-middle-layer/releases/tag/v0.1.0
