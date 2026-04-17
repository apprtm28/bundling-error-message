# PRD: Bundling Issuance Confirmation — Flight Rebooking Conflict Resolution

> **Author**: Order/Bundling PM
> **Status**: DRAFT — Pending cross-team alignment
> **Date**: 2026-04-16
> **Priority**: P0 Launch Blocker (Phase 1, Q2 2026)
> **Affected Teams**: Flight, Order/Bundling, Accommodation
> **Related**: [Problem Statement](../../../references/bundling-ticket-issuance-confirmation/problem.md) | [Bundling Spec §8](../../../references/general/bundling_initiative_spec.md#8-issuance-error-handling--order-lifecycle) | [ADR-001](../../architecture/bundling-issuance-confirmation/adr-001-flight-rebooking-in-bundling.md)

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Goals & Success Metrics](#2-goals--success-metrics)
3. [User Stories](#3-user-stories)
4. [Functional Requirements](#4-functional-requirements)
5. [Non-Functional Requirements](#5-non-functional-requirements)
6. [Edge Cases & Error States](#6-edge-cases--error-states)
7. [Design & UX Considerations](#7-design--ux-considerations)
8. [Technical Considerations](#8-technical-considerations)
9. [Out of Scope](#9-out-of-scope)
10. [Dependencies](#10-dependencies)
11. [Milestones & Timeline](#11-milestones--timeline)
12. [Open Questions](#12-open-questions)

---

## 1. Problem Statement

The Hotel x Flight Bundling product uses **sequential issuance** (`Flight → Hotel`) because Accommodation's opaque pricing — sourced from Expedia — requires hotel booking invalidation if the flight cannot be issued. Expedia unlocks special rate keys only when the hotel is purchased opaquely alongside a confirmed flight.

However, Flight service has an internal rebooking mechanism with a **>30% success rate** that activates when issuance fails. This mechanism:

1. **Does not notify** Order/Bundling service that issuance failed or that rebooking is in progress.
2. On success, **creates a new order ID** to replace the original — incompatible with the L1/L2 bundling order structure.
3. On failure, finally reports issuance failure — but only after an indeterminate delay.

This creates two critical conflicts for bundling:

- **Structural**: Flight's order ID replacement cannot surgically replace only the flight L2 within a composite L1 bundling order. Using the current mechanic would destroy or orphan the hotel L2.
- **Temporal**: The bundling orchestrator interprets the original flight's transitional "failed" status as a signal to abort hotel issuance and refund both items — even while Flight is silently rebooking with >30% chance of success.

**If unresolved**, production scenarios include: stuck orders (orchestrator waiting indefinitely), double-refund inconsistencies (hotel refunded but flight rebook succeeds), and orphaned hotel bookings (flight order ID changes mid-flow).

This is a **P0 launch blocker** — the bundling issuance flow cannot ship without a defined contract between Flight and Bundling for the rebooking scenario.

### Constraint: Why Sequential Issuance Is Non-Negotiable

Sequential issuance is not a design preference — it is an **external contractual requirement**:

- Expedia's opaque rate keys are only valid when the hotel is displayed as a combined price with a flight.
- If the flight is not confirmed, the opaque rate contract is void and the hotel booking must be invalidated.
- Parallel issuance would risk issuing a hotel at an opaque rate for which no valid flight exists.

---

## 2. Goals & Success Metrics

### 2.1 Goals

| # | Goal | Rationale |
|---|------|-----------|
| G1 | Define an unambiguous issuance contract between Flight and Bundling | Eliminates race conditions and structural conflicts |
| G2 | Preserve Flight's rebooking benefit where possible | >30% success rate means real user value at stake |
| G3 | Maintain Expedia opaque pricing contract compliance | Sequential issuance must remain intact |
| G4 | Define timeout and callback behavior for the orchestrator | Orchestrator needs deterministic decision points |

### 2.2 Success Metrics

| Metric | Baseline | Target | Measurement |
|--------|----------|--------|-------------|
| Bundling issuance success rate | N/A (new product; baseline in Month 1) | ≥95% of paid bundling orders fully issued | Order service telemetry |
| Flight issuance-to-callback latency (bundling) | N/A | p95 ≤ T seconds (T = agreed timeout, see [§4.3](#43-timeout--callback-contract)) | Flight service + orchestrator logs |
| Inconsistent order state incidents | N/A | 0 per month | Alerting on L1/L2 linkage integrity checks |
| Hotel issuance not triggered due to false flight failure | N/A | 0 per month | Orchestrator state machine audit log |
| Rebooking recovery rate in bundling context | Standalone: >30% | ≥ standalone rate (if rebooking is supported) OR clearly documented trade-off (if skipped) | Flight service metrics, segmented by `is_bundling` |

---

## 3. User Stories

### US-1: Orchestrator Receives Deterministic Flight Issuance Outcome (P0)

**As** the Bundling orchestrator,
**I want** to receive a single, deterministic issuance outcome (success or failure) from Flight service within a bounded time window,
**So that** I can proceed with hotel issuance or trigger refund without risk of race conditions.

**Acceptance Criteria:**

- Given a bundling order where Flight issuance is requested,
  When Flight service processes the issuance,
  Then it returns exactly one of: `ISSUED`, `FAILED`, or `REBOOKING_IN_PROGRESS` within the agreed timeout T.

- Given Flight returns `ISSUED`,
  When the orchestrator receives the callback,
  Then hotel issuance is triggered using the **same flight L2 order detail ID** (no order ID replacement occurred).

- Given Flight returns `FAILED`,
  When the orchestrator receives the callback,
  Then auto-refund is triggered for **both** flight and hotel items.

- Given the agreed timeout T elapses with no callback from Flight,
  When the orchestrator's timeout fires,
  Then the orchestrator transitions to a defined fallback state (see [§4.3](#43-timeout--callback-contract)).

### US-2: Flight Order ID Integrity in Bundling Context (P0)

**As** the Bundling orchestrator,
**I want** the flight L2 order detail ID to remain unchanged throughout the issuance process (including any rebooking),
**So that** the L1/L2 bundling order structure is never broken.

**Acceptance Criteria:**

- Given a bundling flight issuance request with L2 order detail ID `X`,
  When Flight performs any internal rebooking,
  Then the rebooked ticket is associated with the **original** order detail ID `X` (not a new order ID).

- Given a bundling flight issuance that triggers rebooking,
  When rebooking creates new PNR/ticket data,
  Then the new PNR/ticket data is linked to the original L2 order detail ID `X`,
  And the original order detail is **not** marked as "replaced" or "updated to new booking."

### US-3: Rebooking Transparency to Orchestrator (P1)

**As** the Bundling orchestrator,
**I want** visibility into whether Flight is currently rebooking (if the chosen solution supports it),
**So that** I can make informed decisions about timeout handling and user communication.

**Acceptance Criteria:**

- Given a bundling flight issuance where rebooking is attempted,
  When Flight begins the rebooking process,
  Then either:
  (a) Flight completes rebooking within the issuance timeout and returns a final `ISSUED` or `FAILED` — making the rebooking transparent to the orchestrator, **OR**
  (b) Flight sends an intermediate `REBOOKING_IN_PROGRESS` status to the orchestrator within T₁ seconds, followed by a final `ISSUED` or `FAILED` within T₂ seconds.

- Given the orchestrator receives `REBOOKING_IN_PROGRESS`,
  When the extended timeout T₂ elapses with no final status,
  Then the orchestrator treats it as `FAILED` and triggers refund for both items.

### US-4: User Visibility During Extended Issuance (P2)

**As** a user who has paid for a Flight + Hotel bundle,
**I want** to see an appropriate status while my order is being processed,
**So that** I am not confused by a long issuance wait time.

**Acceptance Criteria:**

- Given a bundling order in `ISSUING` state,
  When the user views their order in My Order,
  Then they see a status indicating issuance is in progress (existing `ISSUING` state).

- Given the issuance takes longer than N seconds (due to rebooking),
  When the user views their order,
  Then they see the same `ISSUING` status (no new user-facing state is introduced in Phase 1).

### US-5: Timeout-Triggered Failure Handling (P0)

**As** the Bundling orchestrator,
**I want** to enforce a maximum wait time for Flight issuance,
**So that** orders do not remain stuck in `ISSUING` state indefinitely.

**Acceptance Criteria:**

- Given a bundling flight issuance request,
  When the total elapsed time exceeds T_max without a terminal response from Flight,
  Then the orchestrator marks flight issuance as `TIMED_OUT`.

- Given the orchestrator marks flight as `TIMED_OUT`,
  When it transitions to refund processing,
  Then it sends a cancellation/abort signal to Flight service to prevent late-arriving rebook success from creating an inconsistent state.

- Given Flight service receives a cancellation signal for an order it already rebooked successfully,
  When Flight processes the cancellation,
  Then Flight cancels/voids the rebooked ticket and confirms cancellation to the orchestrator.

### US-6: Idempotent Issuance Callbacks (P0)

**As** the Bundling orchestrator,
**I want** issuance callbacks from Flight to be idempotent,
**So that** duplicate or late-arriving callbacks do not cause double-processing.

**Acceptance Criteria:**

- Given the orchestrator has already processed a `FAILED` callback for a flight issuance,
  When a delayed `ISSUED` callback arrives (from a late rebook success),
  Then the orchestrator rejects it and logs a reconciliation alert.

- Given the orchestrator has already processed a timeout-triggered failure,
  When a late `ISSUED` callback arrives from Flight,
  Then the orchestrator rejects it, triggers a compensation action (cancel/void the issued ticket via Flight), and logs a reconciliation alert.

---

## 4. Functional Requirements

### 4.1 Issuance Request Contract

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `order_id` | string | Yes | L1 bundling order ID |
| `order_detail_id` | string | Yes | L2 flight order detail ID |
| `is_bundling` | boolean | Yes | Flag indicating this is a bundling issuance request |
| `issuance_timeout_ms` | int64 | Yes | Maximum time the orchestrator will wait before triggering timeout |
| `callback_url` | string | Yes | Orchestrator endpoint for issuance result callback |
| `idempotency_key` | string | Yes | `sha256(order_id + order_detail_id + attempt_sequence)` |

**Requirement FR-1 (P0):** Flight service MUST accept an `is_bundling` flag in the issuance request. This flag determines Flight's behavior regarding internal rebooking (per the chosen solution in [ADR-001](../../architecture/bundling-issuance-confirmation/adr-001-flight-rebooking-in-bundling.md)).

**Requirement FR-2 (P0):** Flight service MUST return the issuance outcome referencing the **original** `order_detail_id` from the request. It MUST NOT create a replacement order or order detail for bundling issuance requests.

### 4.2 Issuance Callback Contract

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `order_id` | string | Yes | L1 bundling order ID (echo from request) |
| `order_detail_id` | string | Yes | L2 flight order detail ID (echo from request, unchanged) |
| `status` | enum | Yes | `ISSUED` \| `FAILED` \| `REBOOKING_IN_PROGRESS` (if solution supports it) |
| `failure_reason` | string | Conditional | Required if `status = FAILED`. Categorized error code. |
| `pnr` | string | Conditional | Required if `status = ISSUED`. May differ from original if rebooking occurred. |
| `idempotency_key` | string | Yes | Echo from request |
| `timestamp` | ISO8601 | Yes | When Flight determined the outcome |

**Requirement FR-3 (P0):** The callback `status` field MUST be one of the defined enum values. The orchestrator MUST NOT accept any other status for bundling issuance.

**Requirement FR-4 (P0):** If `status = REBOOKING_IN_PROGRESS` is supported by the chosen solution, Flight MUST follow it with a terminal status (`ISSUED` or `FAILED`) within the extended timeout T₂.

**Requirement FR-5 (P0):** Callbacks MUST be idempotent using the `idempotency_key`. The orchestrator rejects duplicate callbacks for the same key after a terminal status has been processed.

### 4.3 Timeout & Callback Contract

The orchestrator enforces a bounded wait time for Flight issuance. The specific timeout values depend on the solution chosen (see [ADR-001](../../architecture/bundling-issuance-confirmation/adr-001-flight-rebooking-in-bundling.md)).

**Timeout Model A — Single Timeout (for solutions without intermediate state):**

```
T_max = agreed maximum wait for final ISSUED/FAILED response
```

| Parameter | Proposed Value | Rationale |
|-----------|---------------|-----------|
| `T_max` | TBD (to be agreed with Flight team) | Must accommodate issuance + optional rebooking within bound |

**Timeout Model B — Two-Phase Timeout (for solutions with REBOOKING_IN_PROGRESS):**

```
T₁ = max wait for initial response (ISSUED / FAILED / REBOOKING_IN_PROGRESS)
T₂ = max additional wait after REBOOKING_IN_PROGRESS for final response
T_max = T₁ + T₂
```

| Parameter | Proposed Value | Rationale |
|-----------|---------------|-----------|
| `T₁` | TBD | Normal issuance latency bound |
| `T₂` | TBD | Rebooking latency bound |

**Requirement FR-6 (P0):** On timeout (T_max exceeded), the orchestrator MUST:
1. Transition the bundling order to a failure/refund path.
2. Send a cancellation signal to Flight service.
3. Log the timeout event for alerting and reconciliation.

**Requirement FR-7 (P0):** Flight service MUST handle the cancellation signal gracefully:
- If rebooking is still in progress → abort rebooking.
- If rebooking already succeeded → void/cancel the rebooked ticket and confirm cancellation.
- If issuance already failed → acknowledge (no-op).

### 4.4 Orchestrator State Machine (Bundling Issuance)

```
PAID
 │
 ▼
ISSUING_FLIGHT ──timeout──► FLIGHT_TIMED_OUT ──► REFUND_BOTH
 │                                                    │
 ├── callback: ISSUED ──► ISSUING_HOTEL              ▼
 │                           │                    ORDER_FAILED
 │                           ├── ISSUED ──► ORDER_ISSUED
 │                           ├── FAILED ──► REFUND_HOTEL_ONLY
 │                           └── timeout ──► REFUND_HOTEL_ONLY
 │
 ├── callback: FAILED ──► REFUND_BOTH ──► ORDER_FAILED
 │
 └── callback: REBOOKING_IN_PROGRESS ──► AWAITING_REBOOK_RESULT
                                            │
                                            ├── callback: ISSUED ──► ISSUING_HOTEL
                                            ├── callback: FAILED ──► REFUND_BOTH
                                            └── timeout (T₂) ──► REFUND_BOTH
```

**Requirement FR-8 (P0):** The orchestrator MUST implement this state machine (or the subset applicable to the chosen solution). State transitions MUST be persisted atomically. No state can be skipped.

**Requirement FR-9 (P0):** The `AWAITING_REBOOK_RESULT` state is only applicable if the chosen solution supports `REBOOKING_IN_PROGRESS`. If the solution skips rebooking entirely, this state is omitted.

### 4.5 Late Callback Handling (Compensation)

**Requirement FR-10 (P0):** If the orchestrator receives a `ISSUED` callback after it has already transitioned to `REFUND_BOTH` (due to timeout or prior `FAILED` callback), it MUST:
1. Reject the callback (do not change state).
2. Trigger a compensation action: call Flight to cancel/void the issued ticket.
3. Log a reconciliation event with severity `CRITICAL`.
4. Alert the on-call team.

**Requirement FR-11 (P1):** A daily reconciliation job MUST compare:
- All bundling orders in `ORDER_FAILED` state.
- Against Flight service's actual ticket status for the corresponding L2 order detail IDs.
- Discrepancies trigger auto-remediation (cancel orphaned tickets) or manual review alerts.

---

## 5. Non-Functional Requirements

### 5.1 Performance

| Requirement | Target | Priority |
|-------------|--------|----------|
| Issuance callback latency (Flight → Orchestrator) | p95 < 500ms network latency | P0 |
| Orchestrator state transition latency | p95 < 100ms (within Spanner transaction) | P0 |
| Timeout accuracy | Timer fires within ±1 second of T_max | P1 |

### 5.2 Reliability

| Requirement | Target | Priority |
|-------------|--------|----------|
| Callback delivery guarantee | At-least-once delivery; orchestrator handles duplicates via idempotency key | P0 |
| State machine persistence | Atomic writes to Spanner; no partial state transitions | P0 |
| Cancellation signal delivery | At-least-once with retry (3 attempts, exponential backoff) | P0 |

### 5.3 Observability

| Requirement | Target | Priority |
|-------------|--------|----------|
| Issuance flow tracing | Distributed trace ID spans Flight issuance request → callback → hotel issuance | P0 |
| Timeout alerting | Alert within 60 seconds of timeout event | P0 |
| Late callback alerting | Alert immediately on late `ISSUED` callback after terminal state | P0 |
| Dashboard: bundling issuance funnel | Real-time view of orders by state (ISSUING_FLIGHT, AWAITING_REBOOK, ISSUING_HOTEL, etc.) | P1 |
| Dashboard: rebooking rate in bundling | % of bundling flight issuances that enter rebooking (if supported) vs. standalone | P1 |

### 5.4 Security

| Requirement | Target | Priority |
|-------------|--------|----------|
| Callback authentication | Mutual TLS or HMAC-signed payload between Flight and Orchestrator | P1 |
| Cancellation signal authorization | Only orchestrator can issue cancellation; Flight validates caller identity | P0 |

---

## 6. Edge Cases & Error States

### 6.1 Edge Case Matrix

| # | Scenario | Expected Behavior | Priority |
|---|----------|-------------------|----------|
| EC-1 | Flight returns `ISSUED` but with a **different** order detail ID | Orchestrator rejects callback, logs `CRITICAL`, alerts on-call. Does NOT proceed to hotel issuance. | P0 |
| EC-2 | Flight returns `REBOOKING_IN_PROGRESS` but never sends final status | Orchestrator timeout T₂ fires → `REFUND_BOTH` → cancellation signal to Flight | P0 |
| EC-3 | Orchestrator sends cancellation signal but Flight is unreachable | Retry 3x with exponential backoff. If all fail → log `CRITICAL`, add to daily reconciliation queue. | P0 |
| EC-4 | Flight rebook succeeds but cancellation signal arrives before Flight confirms to orchestrator | Flight processes cancellation (voids ticket). Orchestrator is already in refund path. Consistent state. | P0 |
| EC-5 | Duplicate issuance request (orchestrator retry due to infra error) | Flight uses `idempotency_key` to deduplicate. Returns same result as original. | P0 |
| EC-6 | Hotel issuance timeout after flight success | Unrelated to this PRD — follows existing hotel issuance timeout flow. Flight remains issued. Hotel refunded. | N/A |
| EC-7 | Payment callback delay causes issuance to start late, compressing timeout window | Timeout T_max is measured from issuance request timestamp, not payment time. No compression. | P1 |
| EC-8 | Flight returns `FAILED` with reason `REBOOKING_NOT_ATTEMPTED` (bundling skip scenario) | Orchestrator proceeds with normal `REFUND_BOTH` path. Logged as expected behavior. | P0 |
| EC-9 | Network partition: orchestrator sends issuance request but Flight never receives it | Orchestrator timeout fires. Sends cancellation signal (Flight has nothing to cancel — no-op). Refund both. | P0 |
| EC-10 | Flight issuance succeeds on first attempt (no rebooking needed) | Happy path. `ISSUED` callback → hotel issuance proceeds. No special handling. | P0 |

### 6.2 Failure Reason Taxonomy

Flight MUST categorize failure reasons to enable orchestrator decision-making and analytics:

| Failure Code | Description | Orchestrator Action |
|-------------|-------------|-------------------|
| `INVENTORY_UNAVAILABLE` | No seats available at requested fare | REFUND_BOTH |
| `FARE_EXPIRED` | Fare no longer valid | REFUND_BOTH |
| `REBOOKING_FAILED` | Rebooking was attempted but all alternatives exhausted | REFUND_BOTH |
| `REBOOKING_NOT_ATTEMPTED` | Rebooking skipped (bundling flag, if applicable) | REFUND_BOTH |
| `ISSUANCE_TIMEOUT_INTERNAL` | Flight's internal processing exceeded its own timeout | REFUND_BOTH |
| `AIRLINE_REJECTION` | Airline system rejected the booking | REFUND_BOTH |
| `UNKNOWN` | Unclassified error | REFUND_BOTH + CRITICAL alert |

---

## 7. Design & UX Considerations

### 7.1 User-Facing Impact

This is primarily a **system-to-system contract change**. User-facing impact is limited:

- **No new UI states** in Phase 1. The existing `ISSUING` status in My Order covers the wait period.
- **If the chosen solution extends issuance time** (e.g., waiting for rebooking), the user may experience a longer wait between payment and issuance confirmation. This is bounded by T_max.
- **Notifications**: The existing three notification scenarios (both issued, only flight issued, both failed) remain unchanged. If rebooking is supported, the notification is sent only after the final outcome — the user is not notified about the rebooking attempt itself.

### 7.2 CS / Admin Care Impact

- Admin Care should display the **actual issuance state** (including `AWAITING_REBOOK_RESULT` if applicable) for CS agents troubleshooting stuck orders.
- The "Related Orders" section in Admin Care should flag if a late callback compensation action was triggered.

---

## 8. Technical Considerations

### 8.1 Integration Points

```
┌─────────────────┐     issuance request      ┌─────────────────┐
│                 │ ──────────────────────────► │                 │
│   Bundling      │     callback (status)      │   Flight        │
│   Orchestrator  │ ◄────────────────────────── │   Service       │
│                 │     cancellation signal     │                 │
│                 │ ──────────────────────────► │                 │
└────────┬────────┘                            └─────────────────┘
         │
         │  hotel issuance (on flight ISSUED)
         ▼
┌─────────────────┐
│  Accommodation  │
│  Service        │
└─────────────────┘
```

### 8.2 Data Model Implications

No schema changes to the L1/L2 order structure. The key invariant is:

> The flight L2 `order_detail_id` MUST remain unchanged regardless of rebooking. If Flight internally rebooks to a different PNR/ticket, the new PNR is associated with the original `order_detail_id`.

This may require Flight service to update its internal rebooking logic to support "rebook-in-place" (update existing order detail) vs. "rebook-as-replacement" (create new order, mark old as replaced). The `is_bundling` flag determines which path is taken.

### 8.3 Existing Bundling Spec Alignment

This PRD refines [Section 8 of the Bundling Spec](../../../references/general/bundling_initiative_spec.md#8-issuance-error-handling--order-lifecycle):

| Spec Section | Current Statement | This PRD Adds |
|-------------|-------------------|---------------|
| §8.1 Issuance Sequence | "Flight issuance FAILS → Auto-refund BOTH" | Defines what "FAILS" means when rebooking is involved; timeout bounds; callback contract |
| §8.2 Key Issuance Decisions | "Sequential, not parallel" | Preserved. Adds: Flight callback contract, timeout model, state machine |
| §8.4 Order Status Lifecycle | `PAID → ISSUING → ISSUED / FAILED` | Adds internal orchestrator states (`ISSUING_FLIGHT`, `AWAITING_REBOOK_RESULT`, etc.) not exposed to user |

### 8.4 Solution Options

Three solution approaches are evaluated in detail in [ADR-001](../../architecture/bundling-issuance-confirmation/adr-001-flight-rebooking-in-bundling.md):

| Option | Summary | Rebooking Preserved? | Complexity | Order ID Impact |
|--------|---------|---------------------|------------|-----------------|
| A: Skip Rebooking | Flight skips rebook for bundling; immediate ISSUED/FAILED | No | Low | None |
| B: Synchronous Rebook | Flight rebooks within timeout; returns single final response; no new order ID | Yes | Medium | None (rebook-in-place) |
| C: Two-Phase Orchestration | Orchestrator-aware rebooking with REBOOKING_IN_PROGRESS intermediate state | Yes | High | None (rebook-in-place) |

The chosen solution determines which subset of this PRD's requirements are applicable (see [§4.4](#44-orchestrator-state-machine-bundling-issuance) state machine notes).

---

## 9. Out of Scope

| Item | Rationale |
|------|-----------|
| Hotel issuance timeout/retry changes | Existing Accommodation issuance retry (3 attempts for B2C) is unaffected by this change |
| User-facing rebooking notifications | Phase 1 does not expose rebooking state to users; may be considered in Phase 2 |
| Rebooking behavior for standalone (non-bundling) flight orders | Flight's existing rebooking flow is preserved for standalone orders |
| Parallel issuance (Flight + Hotel simultaneously) | Blocked by Expedia opaque pricing contract requiring sequential confirmation |
| Multi-vertical bundling (3+ items) | Phase 3 scope; this PRD covers Flight + Hotel only |
| Changing the L1/L2 order data model | The solution must work within the existing order hierarchy |
| Flight's internal rebooking implementation details | This PRD specifies the **contract** between Flight and Bundling, not how Flight implements rebooking internally |

---

## 10. Dependencies

| Team | Dependency | Blocking? | Details |
|------|-----------|-----------|---------|
| **Flight** | Implement `is_bundling` flag handling in issuance flow | Yes | Flight must branch behavior based on this flag |
| **Flight** | Support "rebook-in-place" (if Option B or C chosen) | Yes (for B/C) | Rebooked ticket linked to original order detail ID, no new order created |
| **Flight** | Expose cancellation/abort API for bundling issuance | Yes | Orchestrator needs to cancel late-arriving rebook successes |
| **Flight** | Provide rebooking latency distribution data | Yes (for timeout calibration) | p50, p95, p99 of rebooking duration to set T_max / T₂ |
| **Flight** | Provide rebooking success rate data segmented by failure reason | Yes (for decision-making) | Determines feasibility and value of each solution option |
| **Order/Bundling** | Implement orchestrator state machine with timeout | Yes | Core orchestration logic |
| **Order/Bundling** | Implement idempotent callback handler | Yes | Prevent double-processing |
| **Order/Bundling** | Implement late-callback compensation logic | Yes | Cancel orphaned tickets |
| **Accommodation** | No changes required from this PRD | No | Hotel issuance flow unchanged |

---

## 11. Milestones & Timeline

This is a **P0 launch blocker** for Phase 1 (Q2 2026). Timeline depends on the solution chosen:

| Milestone | Option A (Skip) | Option B (Sync Rebook) | Option C (Two-Phase) | Owner |
|-----------|----------------|----------------------|---------------------|-------|
| ADR decision finalized | Week 1 | Week 1 | Week 1 | All teams |
| Flight: `is_bundling` flag + behavior branch | Week 2–3 | Week 2–4 | Week 2–4 | Flight |
| Flight: rebook-in-place support | N/A | Week 3–5 | Week 3–5 | Flight |
| Flight: cancellation/abort API | Week 2–3 | Week 3–4 | Week 3–5 | Flight |
| Orchestrator: state machine + timeout | Week 2–3 | Week 2–4 | Week 2–5 | Order/Bundling |
| Orchestrator: idempotent callback handler | Week 2–3 | Week 2–3 | Week 2–3 | Order/Bundling |
| Orchestrator: late-callback compensation | Week 3 | Week 4 | Week 5 | Order/Bundling |
| Integration testing | Week 4 | Week 5–6 | Week 6–7 | All teams |
| Staging validation | Week 4–5 | Week 6–7 | Week 7–8 | All teams |
| **Total estimated** | **~5 weeks** | **~7 weeks** | **~8 weeks** |  |

---

## 12. Open Questions

| # | Question | Owner | Due Date | Impact |
|---|----------|-------|----------|--------|
| OQ-1 | What is Flight's rebooking latency distribution (p50, p95, p99)? Needed to calibrate T_max and T₂. | Flight PM/Eng | Before ADR decision | Determines timeout values for all options |
| OQ-2 | Can Flight implement "rebook-in-place" (update existing order detail instead of creating new order)? What is the effort estimate? | Flight Eng | Before ADR decision | Determines feasibility of Options B and C |
| OQ-3 | Does Flight currently have a cancellation/abort API for in-progress issuances? If not, what is the effort to build one? | Flight Eng | Before ADR decision | Required for all options (timeout compensation) |
| OQ-4 | Is the >30% rebooking success rate consistent across all failure reasons, or concentrated in specific scenarios (e.g., fare expiry vs. airline rejection)? | Flight PM | Before ADR decision | If concentrated in rare scenarios, Option A impact may be lower than expected |
| OQ-5 | What is the SLA expectation for the user between payment and issuance confirmation for bundling? Does extending this window for rebooking affect conversion or satisfaction? | Order PM + UX Research | Week 2 | Determines acceptable T_max range |
| OQ-6 | If Option A (skip rebooking) is chosen, can we add rebooking support later (Option B or C) without breaking the contract? | All Eng | Week 1 | Reversibility/future-proofing |
| OQ-7 | How does the hotel rate key expiry work at Expedia? Is there a time window after payment during which the rate key remains valid? | Accommodation PM | Week 1 | If rate key has a long validity window, it provides slack for rebooking scenarios |

---

*This PRD will be updated based on the ADR-001 decision outcome and cross-team alignment on timeout values.*
