# ADR-001: Flight Rebooking Behavior in Bundling Issuance Context

## Status

**Proposed** — Pending cross-team decision (Flight, Order/Bundling, Accommodation)

## Date

2026-04-16

## Context

The Hotel x Flight Bundling product uses sequential issuance (`Flight → Hotel`). This sequence is non-negotiable because Accommodation's opaque pricing — sourced from Expedia — requires the hotel booking to be invalidated if the flight cannot be issued. The hotel's special rate key is contractually void without a confirmed flight.

Flight service has an internal rebooking mechanism that activates when issuance fails. This mechanism has a **>30% success rate** — a meaningful recovery path that saves orders that would otherwise fail. However, the mechanism:

1. **Is opaque**: Flight does not notify the Bundling orchestrator that rebooking is in progress.
2. **Creates a new order ID**: On rebook success, a new order replaces the old one — incompatible with the L1/L2 bundling structure where the flight is one component of a composite order.
3. **Has indeterminate latency**: The orchestrator cannot distinguish between "Flight is still trying" and "Flight is stuck."

These behaviors create two critical conflicts:

- **Structural conflict**: The new order ID replaces the entire order, but in bundling, only the flight L2 should be affected. The hotel L2 and L1 wrapper must be preserved.
- **Temporal conflict**: The original flight's transitional "failed" state triggers the orchestrator to refund both items — even while Flight is silently rebooking.

### Forces

- **Rebooking value is high**: >30% recovery rate means skipping rebooking increases bundling failure rate by a meaningful amount.
- **Order structure integrity is non-negotiable**: The L1/L2 model cannot be broken; hotel L2 must always remain linked to its L1.
- **Expedia contract compliance**: Hotel issuance must not proceed until flight is confirmed.
- **Launch timeline pressure**: Phase 1 targets Q2 2026; this is a P0 blocker.
- **Team autonomy**: Flight service owns its internal rebooking logic; changes require Flight team buy-in and capacity.

### Constraints

1. The L1/L2 order data model is fixed — no schema changes.
2. Sequential issuance (`Flight → Hotel`) is fixed — driven by Expedia's opaque pricing contract.
3. The orchestrator must reach a terminal state within a bounded time — no indefinite waits.
4. Late-arriving callbacks must not corrupt order state — idempotency and compensation are required regardless of solution.

---

## Decision

**To be determined** after cross-team discussion. Three alternatives are evaluated below with a recommendation framework.

---

## Alternative A: Skip Rebooking for Bundling Orders

### Description

When Flight receives an issuance request with `is_bundling = true`, it **skips the internal rebooking flow entirely** and immediately returns `ISSUED` or `FAILED` to the Bundling orchestrator.

### Sequence Diagram

```
Orchestrator          Flight Service
    │                      │
    │──issuance request───►│
    │   (is_bundling=true) │
    │                      │── attempt issuance
    │                      │
    │                      │── if fail: DO NOT rebook
    │                      │
    │◄──callback: ISSUED───│  (success)
    │   or                 │
    │◄──callback: FAILED───│  (failure, reason: REBOOKING_NOT_ATTEMPTED)
    │                      │
```

### Pros

- **Simplest implementation**: Flight adds a flag check to skip rebooking; orchestrator handles only two terminal states.
- **Fastest delivery**: Estimated ~5 weeks total. Lowest risk to Phase 1 timeline.
- **No structural changes**: No "rebook-in-place" logic needed in Flight. No new intermediate states in orchestrator.
- **Deterministic latency**: Issuance response time is bounded by Flight's normal issuance latency (no rebooking tail).
- **Clean failure path**: Flight fail → refund both → no ambiguity.

### Cons

- **User impact**: >30% of flight issuance failures that would have been recovered via rebooking will now result in full bundling order failure (both items refunded).
- **Asymmetric experience**: Standalone flight orders get the rebooking safety net; bundling orders do not. Users who book bundles have a worse failure experience.
- **Lost GBV**: Each failed bundling order = lost flight + hotel revenue that may not be recaptured.
- **Reversibility concern**: If we ship this and later want to add rebooking (Options B or C), the orchestrator state machine must be extended — but the issuance contract remains forward-compatible if designed carefully.

### Estimated Impact

If Flight issuance fails X% of the time, and rebooking recovers >30% of those failures:
- Bundling failure rate without rebooking: X%
- Bundling failure rate with rebooking: ~0.7X%
- Additional failed bundling orders per month: 0.3X% × monthly bundling volume

*Exact impact requires Flight team data on issuance failure rate (OQ-1 in PRD).*

---

## Alternative B: Synchronous Rebooking Within Issuance Timeout

### Description

Flight service performs rebooking **within the issuance timeout window** when `is_bundling = true`. From the orchestrator's perspective, it receives a single final response (`ISSUED` or `FAILED`) — it never sees the rebooking attempt. The key behavioral change: Flight performs **rebook-in-place** (updates the existing order detail with new PNR/ticket data) instead of creating a new order ID.

### Sequence Diagram

```
Orchestrator          Flight Service
    │                      │
    │──issuance request───►│
    │   (is_bundling=true) │
    │   (timeout=T_max)    │
    │                      │── attempt issuance
    │                      │── if fail: attempt rebook-in-place
    │                      │   (same order_detail_id, new PNR)
    │                      │── if rebook success: callback ISSUED
    │                      │── if rebook fail: callback FAILED
    │                      │
    │◄──callback: ISSUED───│  (original issuance OR rebook success)
    │   or                 │
    │◄──callback: FAILED───│  (both issuance and rebook failed)
    │                      │
    │                      │
    ├──(timeout T_max)─────┤  if no callback within T_max:
    │                      │
    │──cancel signal──────►│  orchestrator aborts, triggers REFUND_BOTH
    │                      │
```

### Pros

- **Preserves rebooking value**: >30% recovery rate is maintained for bundling orders. No asymmetric user experience.
- **Simple orchestrator**: Only two terminal states (`ISSUED`, `FAILED`). No `REBOOKING_IN_PROGRESS` complexity. State machine identical to Option A.
- **No new intermediate states**: Rebooking is Flight's internal affair — orchestrator just waits for a final answer within T_max.
- **Order ID integrity**: "Rebook-in-place" keeps the original `order_detail_id`. No L1/L2 structural issues.

### Cons

- **Flight must implement "rebook-in-place"**: This is a new capability. Currently, rebooking creates a new order. For bundling, Flight must update the existing order detail with new PNR/ticket data instead. Effort TBD (OQ-2 in PRD).
- **Longer issuance latency**: T_max must accommodate both normal issuance AND rebooking. If rebooking takes, say, 30-60 seconds at p95, the timeout is that much longer. Users wait longer for confirmation.
- **Timeout calibration is critical**: T_max too short → rebooking attempts get cut off, reducing recovery rate. T_max too long → users wait excessively, hotel rate key may approach expiry.
- **Cancellation API required**: If T_max fires while Flight is mid-rebook, the orchestrator must cancel. Flight needs an abort/cancellation endpoint (required for all options, but timing is more sensitive here).

### Key Design Decisions

1. **T_max value**: Must be calibrated from Flight's rebooking latency data (p95 or p99). Requires OQ-1 data.
2. **Rebook-in-place semantics**: Flight updates `order_detail.pnr`, `order_detail.ticket_data`, etc. in Spanner under the **same** `order_detail_id`. The old PNR is voided.
3. **Expedia rate key validity**: If Expedia rate keys expire within T_max, this option is not viable. Requires OQ-7 data.

### Estimated Effort

| Component | Team | Estimate |
|-----------|------|----------|
| `is_bundling` flag handling | Flight | 1 week |
| Rebook-in-place logic (new) | Flight | 2–3 weeks (TBD based on current rebooking architecture) |
| Cancellation/abort API | Flight | 1 week |
| Orchestrator state machine + timeout | Order/Bundling | 2 weeks |
| Idempotent callback + compensation | Order/Bundling | 1 week |
| Integration + staging | All | 2 weeks |
| **Total** | | **~7 weeks** |

---

## Alternative C: Two-Phase Orchestration with Explicit Rebooking State

### Description

Flight service notifies the orchestrator when it enters rebooking, sending an intermediate `REBOOKING_IN_PROGRESS` status. The orchestrator enters an extended wait state with a separate timeout. Flight performs rebook-in-place and sends a final `ISSUED` or `FAILED`.

### Sequence Diagram

```
Orchestrator              Flight Service
    │                          │
    │──issuance request───────►│
    │   (is_bundling=true)     │
    │   (timeout_T1, timeout_T2)│
    │                          │── attempt issuance
    │                          │
    │                          │── if success:
    │◄──callback: ISSUED───────│
    │                          │
    │                          │── if fail, rebook possible:
    │◄──callback: REBOOKING────│  (within T₁)
    │   _IN_PROGRESS           │
    │                          │── attempt rebook-in-place
    │── (start T₂ timer)       │
    │                          │── if rebook success:
    │◄──callback: ISSUED───────│  (within T₂)
    │                          │── if rebook fail:
    │◄──callback: FAILED───────│  (within T₂)
    │                          │
    ├──(timeout T₁)────────────┤  if no initial callback:
    │──cancel signal──────────►│  REFUND_BOTH
    │                          │
    ├──(timeout T₂)────────────┤  if REBOOKING but no final:
    │──cancel signal──────────►│  REFUND_BOTH
    │                          │
    │                          │── if fail, rebook NOT possible:
    │◄──callback: FAILED───────│  (within T₁)
    │                          │
```

### Pros

- **Maximum transparency**: Orchestrator knows exactly what Flight is doing at every stage. Best for observability and debugging.
- **Preserves rebooking value**: Same as Option B — >30% recovery maintained.
- **Optimized timeouts**: T₁ (normal issuance) can be tight. T₂ (rebooking) gets a separate, appropriately sized window. No need for a single large T_max that wastes time when rebooking isn't happening.
- **Future-proof**: The `REBOOKING_IN_PROGRESS` state can be used for user-facing status in Phase 2 ("We're finding you an alternative flight").
- **Selective rebooking**: Flight can decide whether rebooking is even worth attempting for a given failure reason and signal `FAILED` immediately for non-recoverable errors (e.g., airline rejection) vs. `REBOOKING_IN_PROGRESS` for recoverable ones (e.g., fare expired).

### Cons

- **Highest complexity**: Three-state callback model. Two timeout timers. Richer state machine in orchestrator.
- **Longest delivery time**: Estimated ~8 weeks. Highest risk to Phase 1 timeline.
- **Flight must implement two new capabilities**: Rebook-in-place (same as Option B) PLUS intermediate status callback.
- **Testing surface area**: More state transitions = more edge cases to test (e.g., T₁ fires just as `REBOOKING_IN_PROGRESS` is in transit).
- **Network sensitivity**: Two callbacks per issuance (intermediate + final) doubles the callback delivery surface area for at-least-once guarantees.

### Key Design Decisions

1. **T₁ and T₂ values**: T₁ covers normal issuance latency (seconds). T₂ covers rebooking latency (tens of seconds). Both require Flight data.
2. **When to send REBOOKING_IN_PROGRESS vs. FAILED**: Flight must have a classification of recoverable vs. non-recoverable failures. Not all failures should trigger rebooking.
3. **CS visibility**: Admin Care could display "Flight rebooking in progress" for the `AWAITING_REBOOK_RESULT` state, giving CS agents better context.

### Estimated Effort

| Component | Team | Estimate |
|-----------|------|----------|
| `is_bundling` flag handling | Flight | 1 week |
| Rebook-in-place logic (new) | Flight | 2–3 weeks |
| Intermediate REBOOKING_IN_PROGRESS callback | Flight | 1 week |
| Cancellation/abort API | Flight | 1 week |
| Orchestrator state machine (3-state) + dual timeout | Order/Bundling | 2–3 weeks |
| Idempotent callback + compensation | Order/Bundling | 1 week |
| Integration + staging | All | 2 weeks |
| **Total** | | **~8 weeks** |

---

## Comparison Matrix

| Dimension | Option A: Skip Rebook | Option B: Sync Rebook | Option C: Two-Phase |
|-----------|----------------------|----------------------|---------------------|
| **Rebooking preserved** | No | Yes | Yes |
| **Bundling failure rate impact** | +30% of flight failures become order failures | Same as standalone | Same as standalone |
| **Orchestrator complexity** | Low (2 states) | Low (2 states) | High (3+ states, dual timeout) |
| **Flight team effort** | Low (flag check) | Medium (rebook-in-place) | High (rebook-in-place + intermediate callback) |
| **Issuance latency** | Fastest (no rebook wait) | Moderate (single T_max) | Optimal (tight T₁ + adaptive T₂) |
| **Order ID integrity** | N/A (no rebook) | Preserved (rebook-in-place) | Preserved (rebook-in-place) |
| **Observability** | Simple | Moderate (rebook is hidden) | Best (full visibility) |
| **Timeline to ship** | ~5 weeks | ~7 weeks | ~8 weeks |
| **Phase 1 timeline risk** | Low | Medium | High |
| **Future extensibility** | Contract allows adding rebook later | Can add intermediate state later | Already future-proofed |
| **Reversal cost** | Low (add rebook later) | Low (add/remove intermediate state) | Medium (remove complexity) |

---

## Recommendation Framework

The team should choose based on the following data points (referenced as Open Questions in the PRD):

### Choose Option A if:
- Flight team cannot deliver rebook-in-place within Phase 1 timeline.
- Rebooking success rate, when segmented by bundling-relevant failure reasons, is significantly lower than the aggregate >30%.
- The team accepts the GBV loss and plans to upgrade to B or C post-launch.

### Choose Option B if:
- Flight team can deliver rebook-in-place within Phase 1 timeline.
- Rebooking latency (p95) fits within an acceptable T_max that doesn't degrade user experience or risk Expedia rate key expiry.
- The team values simplicity on the orchestrator side (same as Option A from orchestrator's perspective).

### Choose Option C if:
- Flight team can deliver both rebook-in-place AND intermediate callbacks within Phase 1 timeline.
- Observability and CS tooling for rebooking state are high priority.
- The team values the timeout optimization (tight T₁ for non-rebook cases, extended T₂ only when needed).
- There is appetite for future user-facing rebooking status (Phase 2).

### Phased Approach (A → B or C):
If timeline is the primary constraint, **Option A can be shipped for Phase 1 launch** with a commitment to upgrade to Option B or C in Phase 2. The issuance contract (`is_bundling` flag, callback structure, idempotency key, cancellation API) is designed to be forward-compatible — adding `REBOOKING_IN_PROGRESS` and rebook-in-place later does not break the existing contract.

---

## Consequences (Common to All Options)

### Positive (All Options)

- Bundling issuance flow has a deterministic, bounded state machine — no stuck orders.
- L1/L2 order structure integrity is guaranteed — no orphaned hotel bookings.
- Late callback compensation prevents inconsistent states.
- Idempotent callbacks prevent double-processing.
- The issuance contract is formally defined — no implicit assumptions between Flight and Bundling.

### Negative (All Options)

- Flight service must respect the `is_bundling` flag and alter its behavior — introducing a conditional code path.
- A cancellation/abort API must be built — new surface area to maintain and test.
- Timeout values require empirical calibration and may need adjustment post-launch.

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| T_max is miscalibrated (too short → cuts off valid rebooks; too long → users wait excessively) | Medium | High | Use Flight's rebooking latency p99 as T_max with 20% buffer. Monitor and adjust post-launch. |
| Flight's rebook-in-place has bugs, linking wrong PNR to order detail | Low | Critical | Thorough integration testing. Daily reconciliation job validates PNR ↔ order_detail_id linkage. |
| Late callback arrives after compensation already cancelled the ticket | Medium | Medium | Compensation logic is idempotent. Flight confirms cancellation before marking void. |
| Expedia rate key expires during extended issuance wait (Options B/C) | Low–Medium | High | Measure rate key TTL (OQ-7). Set T_max < rate key TTL with margin. |

## Reversal Cost

- **Option A → B**: Low. Add rebook-in-place in Flight. Orchestrator contract unchanged (still two terminal states).
- **Option A → C**: Medium. Add rebook-in-place + intermediate callback. Extend orchestrator state machine.
- **Option B → C**: Low. Add intermediate callback. Extend orchestrator with `AWAITING_REBOOK_RESULT` state.
- **Option C → B**: Low. Remove intermediate callback. Simplify orchestrator state machine.
- **Option B or C → A**: Low. Remove rebook-in-place. Flag check already exists.

---

*This ADR will be updated with the team's decision after the cross-team discussion. The chosen option will move to `Accepted` status and the PRD requirements will be finalized accordingly.*
