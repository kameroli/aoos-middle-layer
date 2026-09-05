# AOSS Vendor Adapter Interface

**Version:** 0.1.0 (Draft for Comment)

Adapters are the only place where vendor-specific and internal-system-specific vocabulary is allowed to exist. Everything past the adapter boundary — statuses, errors, outcomes, events — speaks AOSS. This document defines the adapter contract, mapping rules, and the requirements every adapter implementation MUST meet. RFC 2119 keywords apply as in `SPEC.md`.

## 1. Adapter contract

An adapter is a stateless (or with externally stored state) component with four responsibilities:

1. **Request normalization.** Translate an AOSS-domain request (e.g., "verify this applicant's identity," "evaluate this credit application") into the vendor's wire format, including authentication, field mapping, and any vendor-required session setup. The adapter receives only the fields it needs; it MUST NOT be handed the full application if a subset suffices.
2. **Response normalization.** Translate the vendor's response into exactly one of:
   - a **status signal** — a proposed AOSS transition event (e.g., `decision.approved`, `information.requested`) with supporting detail;
   - a **normalized error** — an `ErrorDetail` with a canonical category, stable code, and retryability flag; or
   - an **indeterminate result** — an explicit "unknown, reconcile" signal (e.g., after a timeout), never a guess.
3. **Reason mapping.** Map vendor decline/refer reasons into the adopter's versioned reason catalog (see `SPEC.md` Section 7). An unmapped vendor reason MUST NOT pass through raw: the adapter maps it to a catalog entry flagged `unmapped`, emits an operational alert, and the application routes to manual review rather than auto-decline on unrecognized grounds.
4. **Observability.** Propagate the correlation ID into every vendor call (via the vendor's trace/reference field where one exists, and in adapter logs regardless), record request/response timing metrics, and redact PII from anything it logs (`SPEC.md` Section 6.3).

The adapter interface, expressed neutrally:

```
interface DecisionAdapter<Req, Signal> {
  // Idempotent where the vendor allows it; see Section 4.
  invoke(request: Req, context: CallContext): AdapterResult<Signal>
  // Health signal used by circuit breakers and routing.
  probe(): AdapterHealth
}

CallContext  = { applicationId, correlationId, idempotencyKey, deadline }
AdapterResult = StatusSignal | NormalizedError | Indeterminate
```

Adapters MUST be deterministic mappers: the same vendor response always yields the same AOSS result. Mapping tables (Sections 2–3) are configuration, versioned alongside the adapter, and covered by contract tests (`TESTING.md`).

**Boundary rules.** Adapters MUST NOT: apply status transitions themselves (they propose signals; the middle layer's state machine applies them), invent statuses or error categories, retry beyond the policy handed to them in `CallContext`, or persist PII outside the middle layer's designated stores.

## 2. Example mapping: generic identity-verification (IDV) vendor

Illustrative only; every real vendor's vocabulary differs. Vendor terms below are generic placeholders.

| Generic vendor response                     | AOSS mapping                                                                               | Notes                                                                                                        |
| ------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| `verified` / match score above threshold    | StatusSignal: contributes to `decision.approved` path (IDV check passed)                   | The adapter reports "check passed"; the middle layer decides overall status once all required checks report. |
| `not_verified` — data mismatch              | NormalizedError: `identity_verification_failure`, code `idv.data_mismatch`, retryable=true | Typically routes the application to `needs_more_information` with a `confirm_detail` next step.              |
| `document_unreadable` / `image_quality_low` | NormalizedError: `document_quality`, code `idv.image_quality`, retryable=true              | Routes to `needs_more_information` with a `provide_document` next step carrying capture guidance.            |
| `document_expired`                          | NormalizedError: `document_quality`, code `idv.document_expired`, retryable=true           | Applicant supplies a current document.                                                                       |
| `refer` / `manual_review`                   | StatusSignal: hold in `under_review`, sub_status `review.manual_idv`                       | No applicant-facing change; operational queue entry created.                                                 |
| HTTP 429                                    | NormalizedError: `vendor_unavailable`, code `idv.rate_limited`, retryable=true             | Backoff per Section 4; honor any Retry-After equivalent.                                                     |
| HTTP 5xx / connection failure               | NormalizedError: `vendor_unavailable`, code `idv.upstream_error`, retryable=true           | Counts toward circuit breaker.                                                                               |
| No response within deadline                 | Indeterminate → NormalizedError: `timeout`, code `idv.timeout`, retryable=true             | Reconcile before re-invoking (Section 4).                                                                    |
| HTTP 4xx (malformed request)                | NormalizedError: `internal_error`, code `idv.request_rejected`, retryable=false            | A 4xx from a vendor is an adapter/integration bug, not an applicant problem; alert engineering.              |

## 3. Example mapping: generic decision/underwriting path

| Generic vendor/engine response                   | AOSS mapping                                                                                             | Notes                                                                                                                                              |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `approve`                                        | StatusSignal: `decision.approved`                                                                        | For deposit products, provisioning then drives `account.opened`.                                                                                   |
| `decline` + reason codes                         | StatusSignal: `decision.declined` + reasons mapped into the adopter's catalog with vendor rank preserved | Every vendor reason code MUST have a catalog mapping (Section 1.3). Ranked, specific reasons populate `Outcome.reasons[]` per `SPEC.md` Section 7. |
| `decline` + unrecognized reason code             | Route to manual review; alert                                                                            | Never auto-communicate an unmapped reason.                                                                                                         |
| `conditional` / `refer`                          | StatusSignal: `information.requested` or hold in `under_review`, per condition type                      | Conditions the applicant can satisfy become `NextStep` entries; internal conditions become review-queue items.                                     |
| `counteroffer` (engine proposes different terms) | Out of scope for v0.1 core; hold in `under_review` for operator handling                                 | Candidate for a future extension; feedback welcome.                                                                                                |
| Engine unavailable / timeout                     | `vendor_unavailable` / `timeout` as in Section 2                                                         | The application MUST remain `under_review`; retry exhaustion never becomes a decline.                                                              |

## 4. Retry and idempotency rules

- **Idempotency first.** Where the vendor supports an idempotency or client-reference key, the adapter MUST send one derived deterministically from `(applicationId, operation, attempt-group)` — stable across retries of the same logical call. Where the vendor does not, the adapter MUST implement its own replay guard (record the outbound call durably before sending; on retry after an indeterminate result, reconcile via the vendor's query/lookup interface before re-invoking).
- **Backoff.** Retries use exponential backoff with jitter within the `deadline` in `CallContext`; the middle layer, not the adapter, owns the overall retry budget.
- **Never retry:** responses mapped to non-retryable errors, and any mutating call after an indeterminate result until reconciliation has established that the original did not take effect.
- **Circuit breaking.** Adapters SHOULD expose failure-rate signals via `probe()` so the middle layer can open a circuit, keep applications safely in `under_review`, and alert operations instead of hammering a degraded vendor.
- **No status regression.** Nothing an adapter returns may move an application backward or to a terminal state except through the middle layer's state machine validation.

## 5. Sandbox adapter

The conformance suite specified in TESTING.md relies on a sandbox adapter: a simulated vendor implementing the full adapter contract with scripted behavior. It MUST support, selectable per call via test configuration or magic input values:

- each mapped happy-path response (approve, decline-with-reasons, refer, verified, not-verified, document-quality failures);
- fault injection: timeout, 5xx, connection reset, rate limiting, malformed response body, partial response (syntactically valid but missing required fields);
- duplicate-delivery simulation (same vendor response delivered twice) to exercise replay guards;
- configurable latency, to exercise deadlines and time-in-status metrics.

Adopters SHOULD run their client and middle-layer integration tests against the sandbox adapter before any vendor sandbox, and SHOULD keep a sandbox scenario for every mapping-table row so the tables in Sections 2–3 are executable, not just documentation.

---

_v0.1 draft for comment. Feedback is particularly requested on the counteroffer gap (Section 3) and on the minimum required fault set for the sandbox adapter._
