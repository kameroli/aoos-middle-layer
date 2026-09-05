# AOSS Testing Approach

**Version:** 0.1.0 (Draft for Comment)

This document specifies how conformance to AOSS is to be verified: the contract tests, state-machine invariants, and adapter checks an implementation MUST pass to claim conformance. The executable conformance suite ships with the reference implementations (Roadmap, phase 2); until then, this document is the normative definition of what that suite must cover. RFC 2119 keywords apply as in SPEC.md.

## 1. Contract testing

The `openapi.yaml` document is the contract; both sides test against it, not against each other's implementations.

- **Provider side (middle layer):** schema validation of every response against the OpenAPI schemas in CI; enum enforcement (no status or error category outside the canonical sets ever serialized); required-field checks per status (e.g., `reasons[]` present and non-empty when `status = declined`; at least one `next_steps[]` entry when `status = needs_more_information`).
- **Consumer side (SDKs, adopter backends):** consumer-driven contract tests pinning the interactions each client actually uses, so a provider change that breaks a real consumer fails in the provider's build, not in production.
- **Compatibility gate:** a golden set of recorded request/response pairs per spec version. A candidate release MUST reproduce byte-compatible behavior for the current MINOR version and MUST tolerate unknown optional fields, unknown `sub_status` values, and unknown `next_step.type` values (see `SPEC.md` Section 8).

## 2. State machine verification

- **Transition table tests:** every permitted transition in `SPEC.md` Section 3.4 has a passing test; every non-permitted (from, to) pair has a test asserting rejection plus an error log with correlation ID.
- **Replay invariant:** for generated event sequences (property-based testing is well suited here), replaying an application's `StatusEvent` log MUST reproduce its current status, and applying any recorded event twice MUST NOT produce a second state change or duplicate event.
- **Terminality:** no operation — API call, adapter signal, or administrative action outside the defined event path — may move an application out of `declined`, `opened`, `expired`, or `withdrawn`, or out of `approved` except via `account.opened` for deposit products.

## 3. Adapter conformance checklist

Every adapter (including the sandbox adapter in `ADAPTERS.md` Section 5) MUST pass:

- [ ] Every row of its mapping table has an executable test: vendor response fixture in, expected AOSS signal or `ErrorDetail` out.
- [ ] Unmapped vendor decline reason routes to manual review with an `unmapped` catalog entry and an operational alert — never a raw pass-through, never an auto-decline.
- [ ] Same vendor response twice → identical AOSS result (deterministic mapping) and no duplicate state change (replay guard).
- [ ] Correlation ID present on every outbound vendor call and every adapter log line; no PII in any adapter log or metric (verified by scanning fixtures containing sentinel PII values).
- [ ] Retryability flags match `SPEC.md` Section 4.2 for every produced error.
- [ ] Idempotency key or replay guard exercised: retry after an injected indeterminate result triggers reconciliation, not a blind re-invoke.
- [ ] Deadline respected: the adapter returns an indeterminate/timeout result within the `CallContext` deadline rather than blocking past it.

## 4. Integration and failure-mode tests

Run against the sandbox adapter with fault injection; the same suite SHOULD later run against vendor sandboxes where available.

- **Vendor timeout:** application remains `under_review`; a `timeout` error event is recorded; reconciliation runs before any re-invoke; retry exhaustion alerts operations and never produces a terminal status.
- **Vendor unavailable / 5xx / rate limiting:** backoff with jitter observed; circuit opens under sustained failure; applications hold their current status; no `vendor_unavailable` detail leaks into applicant-facing fields.
- **Partial response:** syntactically valid vendor payload missing required fields maps to a normalized error (never a partial status signal, never a crash), with the offending payload redacted in logs.
- **Duplicate submission:** same `Idempotency-Key` + same payload returns the original result with no second application; same key + different payload returns `duplicate_submission`; concurrent duplicate requests (race) still yield exactly one application.
- **Duplicate vendor delivery:** the same vendor callback/response delivered twice produces one state change and one `StatusEvent`.
- **Webhook delivery:** consumer receives at-least-once delivery; deduplication by `event_id` verified; out-of-order delivery handled via `sequence`; a failed endpoint triggers redelivery with backoff, and the `/events` endpoint reconciles the gap.
- **Information round-trip:** `information.requested` → applicant submits via `/information` → `information.provided` → review resumes; iteration count appears in metrics; expiry sweep (where `expired` is implemented) fires only after the configured window and produces exactly one `application.expired` event.

## 5. Outcome quality checks

Beyond schema validity, CI SHOULD lint outcome fixtures: declined outcomes have ranked, catalog-backed reasons with specific descriptions; applicant-facing text fields contain no vendor names, internal system names, stack traces, or sentinel PII; every `needs_more_information` outcome has at least one actionable next step.

---

`*v0.1 draft for comment.`
