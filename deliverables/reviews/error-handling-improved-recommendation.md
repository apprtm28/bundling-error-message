# Bundling Error Aggregation — Improved Recommendation v2

> **Status**: RECOMMENDATION — Synthesized from Brainstorming v1 + Michelle Proposal
> **Problem Owner**: Agung Putra Pratama (Order PM)
> **Date**: 2026-04-16
> **Inputs**: `error_aggregation_brainstorm.md`, `references/Michelle-proposal.md`
> **Improvement Type**: Structural upgrade to initial brainstorming

---

## What Changed From v1

This document improves the brainstorming (v1) by incorporating the Michelle's structural contributions while preserving all implementation depth. Key changes:

| Area | v1 (Brainstorming) | v2 (This Document) | Source |
|---|---|---|---|
| Error classification | Ad-hoc by error code/category | 5-tier hierarchy with canonical mapping | Michelle framework + non-200 extension |
| Display timing | All errors collected, then displayed | Both verticals evaluated in parallel; display fully batched after parallel eval completes (flight T3 → price → accom T3) | Parallel eval model (ADR-008) |
| Non-business error handling | "Cannot continue → reload" | Retry-before-display with limits | Michelle retry pattern |
| Scenario coverage | 8 named scenarios | Full cross-product matrix (5×6 = 30 cells) | Michelle matrix approach + expansion |
| Non-200 HTTP errors | Not addressed | Explicit 5th tier with handling | Gap identified in review |

All ADRs, implementation details, edge cases, API structures, and Approach E + B-lite from v1 are **preserved and incorporated**.

---

## 1. Error Classification Tiers (NEW)

### 1.1 The 5-Tier Framework

Every error from both verticals is classified into exactly one tier. Tiers are ordered by **handling priority** — higher tiers are resolved before lower tiers are evaluated.

| Tier | Name | HTTP Status | Description | Handling Priority |
|:---:|---|---|---|:---:|
| T0 | **Infrastructure Error** | Non-200 | Service unreachable, timeout, 5xx, malformed response, circuit breaker open | 1 (highest) |
| T1 | **System/Non-Business Error** | 200 | Application-level system failure: cart not found, session expired, data parsing error | 2 |
| T2 | **Inventory Error** | 200 | Product unavailable: sold out, fare exhausted, all rooms booked | 3 |
| T3 | **Non-Price Business Error** | 200 | Business rule change that doesn't involve price: baggage, CP, meal, passenger validation | 4 |
| T4 | **Price Error** | 200 | Price change (up or down) from either vertical | 5 (lowest — held until both verticals pass T0-T2) |

### 1.2 Classification Rules

**Priority escalation**: If a response contains errors from multiple tiers, the HIGHEST tier determines the handling path. Example: if accom returns a combination error with price (T4) + CP change (T3), the response is classified as T4 for price handling but T3 items are still extracted and displayed.

**"Successful" definition** (adopted from Michelle): A vertical's final check is successful if and only if:
- No T0 errors (infrastructure is reachable)
- No T1 errors (application is functioning)
- No T2 errors (inventory is available)

Price errors (T4) do NOT block "success" — they are held and aggregated across both verticals.

### 1.3 Flight Error → Tier Mapping (Phase 1 Only)

| Tier | Category | Error Codes | Impact |
|:---:|---|---|---|
| T0 | INFRASTRUCTURE | HTTP 5xx, network timeout, connection refused, malformed JSON | Cannot proceed |
| T1 | SYSTEM_TECHNICAL | BOOKING_UNCOMPLETED_PARAMETER_DATA, BOOKING_CREATE_ORDER_FAILED, BOOKING_ORDER_CORE_FAILED, ERROR_DEFAULT, BOOKING_FAILED, RUNTIME_ERROR, NAMING_RULE_DATA_NOT_EXIST, RESELLER_INFO_DATA_NOT_EXIST, BOOKING_ORDER_UPDATE_ORDER_FAILED, BOOKING_ORDER_CREATE_ORDER_FAILED | Cannot proceed |
| T1 | BOOKING_SESSION_CART | CART_NOT_EXIST | Cannot proceed |
| T1 | QUEUE_TIMEOUT | BOOKING_FLOODING_VALIDATION, BOOKING_FLOODING_KEY_INVALID | Cannot proceed |
| T2 | AIRLINE_COMMUNICATION | FLIGHT_CART_FAILED | Disruptive → SRP |
| T2 | AVAILABILITY_SEAT_FARE | NO_SEAT, BOOKING_INTEGRATOR_NO_SEAT | Disruptive → SRP |
| T2 | AVAILABILITY_SEAT_FARE | FARE_ALREADY_SOLD_OUT, FARE_ALREADY_SOLDOUT | Disruptive → PRP/SRP |
| T2 | BOOKING_DOUBLE_LIMIT | BOOKING_ORDER_LIMIT_RESTRICTED | Disruptive → orderPage |
| T2 | AUTH_LOGIN | LOGIN_REQUIRED, B2B_LOGIN_NEEDED | Disruptive → loginPage |
| T3 | AIRLINE_COMMUNICATION | BOOKING_INTEGRATOR_FAILED | Semi-disruptive → retry/SRP |
| T3 | BOOKING_DOUBLE_LIMIT | DOUBLE_BOOKING_ERROR | Semi-disruptive → orderPage or paymentPage |
| T3 | QUEUE_TIMEOUT | BOOKING_TIME_OUT | Semi-disruptive → SRP or stay |
| T3 | ADDONS_BAGGAGE_CHANGE | INCLUSIVE_BAGGAGE_CHANGE, INCLUSIVE_BAGGAGE_CHANGE_SINGLE_SCHEDULE, INCLUSIVE_BAGGAGE_CHANGE_MULTI_SCHEDULE | Non-disruptive → paymentPage |
| T3 | PASSENGER_DATA_VALIDATION | NAMING_RULE_NAME_ONE_CHARACTER, VALIDATION_PASSENGERIDENTITY_NUMBER_IS_DUPLICATE, VALIDATION_PASSENGER_NAME_IS_DUPLICATE, NAMING_RULE_LAST_NAME_VALIDATION, NAMING_RULE_NAME_IS_TOO_LONG, FORM_VALIDATOR_ERROR | Non-disruptive → bookingFormPage |
| T3 | PASSENGER_RULES_RELATIONSHIP | VALIDATION_INFANT_MORE_THAN_ADULT, ADULT_BELOW_SIXTEEN_YEARS, VALIDATION_ADULT_PAX_DOB_NOT_VALID | Non-disruptive → bookingFormPage/SRP |
| T4 | PRICE_CHANGE_TICKET | PRICE_CHANGE_ADJUSTMENT, PRICE_CHANGE_ADJUSTMENT_UP, PRICE_CHANGE_ADJUSTMENT_DOWN | Price change → suppress & aggregate |
| T4 | PRICE_CHANGE_TICKET | PRICE_CHANGE_AIRLINES, PRICE_CHANGE_AIRLINES_UP, PRICE_CHANGE_AIRLINES_DOWN | Price change → suppress & aggregate |
| T4 | PRICE_CHANGE_TICKET | PRICE_CHANGE_GENERAL, PRICE_CHANGE_GENERAL_UP, PRICE_CHANGE_GENERAL_DOWN | Price change → suppress & aggregate |

### 1.4 Accommodation Error → Tier Mapping

| Tier | Error Type | Bundling Relevance | Impact |
|:---:|---|---|---|
| T0 | HTTP 5xx, network timeout, malformed response | Always relevant | Cannot proceed |
| T1 | Michelleal app error (if accom returns BUSINESS_ERROR with system-level errorCode) | Always relevant | Cannot proceed |
| T2 | All rooms sold out | Relevant | Disruptive → Bundling SRP |
| T3 | Cancellation policy flag change | Relevant; Separated | Non-disruptive → pass through |
| T3 | Cancellation policy detail change | Relevant; Separated | Non-disruptive → pass through |
| T3 | Meal plan/pax change | Relevant; Separated | Non-disruptive → pass through |
| T3 | Value-added benefit change | Relevant; Separated | Non-disruptive → pass through |
| T3 | Insurance error | Relevant; Separated | Non-disruptive → pass through |
| T4 | Price change (standalone) | Relevant; Combined | Price change → suppress & aggregate |
| T4+T3 | Combination error (price + non-price) | **THE LOOPHOLE** | Strip T4 items, pass T3 items through |
| — | Room capacity change | **Not relevant** for bundling (pax managed by bundling) | Skip |
| — | Travel policy change | **Not relevant** (B2B only, Phase 1 = B2C) | Skip |
| — | Payment method change | **Not relevant** (PAH excluded from bundling) | Skip |

---

## 2. Orchestration Flow (UPDATED — Parallel Evaluation)

Parallel evaluation with cancel-fast on flight terminal errors, sequential display.

```
User clicks "Continue to Payment" on Booking Form
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ Bundling BE: Validation Orchestration                       │
│                                                             │
│ ┌─── STEP 1: EVALUATE (parallel) ────────────────────────┐│
│ │                                                          ││
│ │  ┌─────────────────────────┐ ┌──────────────────────────┐││
│ │  │ Call Flight Service      │ │ Call Accom Service       │││
│ │  │ (independent retries:    │ │ (independent retries:    │││
│ │  │  T0 → 2×, backoff 1s/3s │ │  T0 → 2×, backoff 1s/3s │││
│ │  │  T1 → 1×, fixed 2s)     │ │  T1 → 1×, fixed 2s)     │││
│ │  └───────────┬─────────────┘ └────────────┬─────────────┘││
│ │              │                             │              ││
│ │              └─────────┬───────────────────┘              ││
│ │                        │ Both results available           ││
│ └────────────────────────┼──────────────────────────────────┘│
│                          ▼                                   │
│ ┌─── STEP 2: GATE (flight priority) ────────────────────┐  │
│ │                                                          ││
│ │ Flight T0/T1/T2/T3-fix/T3-back (TERMINAL)?              ││
│ │   → Cancel pending accom call. Discard accom result.     ││
│ │   → Show flight error immediately:                       ││
│ │     T0: "Something went wrong. Please try again."        ││
│ │     T1: Flight system error (remapped → BF)              ││
│ │     T2: Flight inventory error (remapped → SRP)          ││
│ │     T3-fix: Validation error (user fixes on BF)          ││
│ │     T3-back: User redirected to SRP                      ││
│ │   → STOP.                                                ││
│ │                                                          ││
│ │ Flight T3-ok / T4 / SUCCESS (PROCEED)?                   ││
│ │   → Save flight data (T3-ok message, T4 price, or OK)   ││
│ │   → Await accom result → proceed to Step 3.             ││
│ └──────────────────────────────────────────────────────────┘│
│                                                             │
│ ┌─── STEP 3: CLASSIFY ACCOM ─────────────────────────────┐│
│ │ (Only reached if flight passed gate)                     ││
│ │                                                          ││
│ │ Accom T0 (Infrastructure)?                               ││
│ │   → Discard any held flight T4. Show accom infra error.  ││
│ │                                                          ││
│ │ Accom T1 (System)?                                       ││
│ │   → Discard any held flight T4. Show accom system error. ││
│ │                                                          ││
│ │ Accom T2 (Inventory — all rooms sold out)?               ││
│ │   → Discard any held flight T4.                          ││
│ │     Remap action → Bundling SRP.                         ││
│ │                                                          ││
│ │ Accom T3 only (non-price changes)?                       ││
│ │   → Queue for display.                                   ││
│ │                                                          ││
│ │ Accom T4 standalone (price change only)?                 ││
│ │   → SUPPRESS. Save: accom_old_price, accom_new_price.   ││
│ │                                                          ││
│ │ Accom T4+T3 (Combination error: price + non-price)?      ││
│ │   → Apply "Filter & Wrap" (Approach E + B-lite):         ││
│ │     1. Extract price via priceData field                  ││
│ │     2. Strip items with changeType === "price"            ││
│ │     3. Rewrite popup.body to exclude "price"              ││
│ │     4. Update tracker eventDescription                    ││
│ │     5. Save: accom_old_price, accom_new_price             ││
│ │     6. Queue remaining T3 items for display               ││
│ │                                                          ││
│ │ Accom SUCCESS (no error)?                                ││
│ │   → Proceed to display.                                  ││
│ └──────────────────────────────────────────────────────────┘│
│                                                             │
│ ┌─── STEP 4: DISPLAY (sequential) ───────────────────────┐│
│ │                                                          ││
│ │ 4a. Flight T3-ok (if captured):                          ││
│ │   → Display flight non-price message.                    ││
│ │     User acknowledges.                                   ││
│ │                                                          ││
│ │ 4b. Price aggregation (if any T4 captured):              ││
│ │   → If price INCREASED (net positive delta):             ││
│ │     old_opaque = (flight_old + accom_old) / pax          ││
│ │     new_opaque = (flight_new + accom_new) / pax          ││
│ │     Display: "Your bundled price has been updated         ││
│ │       from IDR X/pax to IDR Y/pax"                      ││
│ │     Button: "OK, Got It" → paymentPage                   ││
│ │                                                          ││
│ │   → If price DECREASED (net negative delta):             ││
│ │     SILENT PASS-THROUGH. No message shown.               ││
│ │     User sees updated lower price on payment page.       ││
│ │                                                          ││
│ │   → If net zero change:                                  ││
│ │     SILENT PASS-THROUGH. No message shown.               ││
│ │                                                          ││
│ │ 4c. Non-price accom errors (queued T3):                  ││
│ │   → Display after price message (if any)                 ││
│ │   → Use accom's native UI (bottom sheet, etc.)           ││
│ │   → For filtered combination errors: bottom sheet shows  ││
│ │     only non-price changes (safe, no price leak)         ││
│ │                                                          ││
│ │ 4d. Redirect to Payment Page                             ││
│ └──────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Display Sequence (UPDATED — Parallel Evaluation)

Both verticals are evaluated in parallel. Display is fully batched after evaluation completes.

| Step | Timing | What's Shown | Condition |
|:---:|---|---|---|
| 1 | After parallel eval (gate) | Flight T0/T1/T2/T3-fix/T3-back error | If flight terminal → cancel accom, show immediately → STOP |
| 2 | After parallel eval (classify) | Accom T0/T1 error | If accom infra/system fails after retries → STOP |
| 3 | After parallel eval (classify) | Accom T2 error | If accom inventory unavailable → STOP (discard flight T4) |
| 4 | After parallel eval (display) | Flight T3-ok message | If flight T3-ok was captured, shown first in display sequence |
| 5 | After parallel eval (display) | Opaque price change message | If net price increase from T4 (flight and/or accom) |
| 6 | After price message (display) | Accom T3 non-price changes | Queued accom non-price errors shown after price |
| 7 | After all errors shown | Redirect to Payment Page | User has acknowledged all changes |

---

## 4. Retry Semantics (NEW)

| Error Tier | Retry Strategy | Max Retries | Backoff | User-Visible? |
|---|---|:---:|---|---|
| T0 (Infrastructure) | Automatic, before error display | 2 | Exponential: 1s → 3s | No (silent retry) |
| T1 (System/Non-Business) | Automatic, before error display | 1 | Fixed: 2s | No (silent retry) |
| T2 (Inventory) | No automatic retry | 0 | — | Redirect to SRP |
| T3 (Non-Price Business) | No automatic retry | 0 | — | Show error immediately |
| T4 (Price) | No retry (price is valid, just changed) | 0 | — | Aggregate and show |

**Timeout budget**: Total orchestration must complete within 30s. If retries would exceed this, fail fast and show error.

---

## 5. Solution Approach: E + B-lite ("Filter & Wrap") — PRESERVED

All implementation details from v1 Section 7.4 remain unchanged. Summarized here for completeness:

**Layer 1: Price Containment** — Bundling BE intercepts all T4 errors, computes combined opaque delta.

**Layer 2: Non-Price Pass-Through** — Bundling BE strips price from combination errors using `changeType` field, passes non-price errors through with action remapping.

**Minimal ask from Accom team** (unchanged):
1. `changeType` field on `newRatePlanChanges.popup.details[].details[]`
2. `priceData` object with `{ oldTotalAAT, newTotalAAT, currency }`

**Fallback if Accom declines** (ADR-004): Suppress "See changes" button, show generic banner from `eventDescription` field.

---

## 6. Action Button Remapping — PRESERVED

| Original Action | Bundling Remap |
|---|---|
| Flight: `searchResultPage` | → Bundling SRP |
| Flight: `productReviewPage` | → Bundling SRP (with "Change flight" auto-triggered) |
| Flight: `bookingFormPage` | → Bundling Booking Form (stay) |
| Flight: `paymentPage` | → Bundling Payment Page |
| Flight: `submitBooking` | → Re-trigger Bundling validation |
| Flight: `orderPage` | → My Order page |
| Flight: `loginPage` | → Login page (passthrough) |
| Accom: `NAVIGATE_PREV_PAGE` | → Bundling SRP (with hotel section, flight preserved) |
| Accom: `CHANGE_DATES` | → Bundling SRP (with date picker open) |
| Accom: `NAVIGATE_PAYMENT_PAGE` | → Bundling Payment Page |
| Accom: `REFRESH` | → Refresh Bundling Booking Form |
| Accom: `NONE` | → Dismiss dialog |
| Accom: `OPEN_RATE_PLAN_CHANGES_PREBOOK` | → Open filtered bottom sheet (price stripped) |
| Accom: `OPEN_RATE_PLAN_CHANGES_BOOK` | → Open filtered bottom sheet (price stripped) |

---

## 7. ADRs — PRESERVED + NEW

### ADR-001 through ADR-004: From v1 (ADR-001 superseded by ADR-008)

See `error_aggregation_brainstorm.md` Sections 10.1–10.4. **Note:** ADR-001 (Sequential Validation) is superseded by ADR-008 (Parallel Evaluation with Sequential Display).

### ADR-005: 5-Tier Error Classification (NEW)

- **Status:** Proposed
- **Context:** v1 brainstorming classified errors ad-hoc by error code. Michelle proposal introduced a clean 4-tier hierarchy. Non-200 HTTP errors were unaddressed by both documents.
- **Decision:** Adopt a 5-tier error classification (T0-T4) that maps every error code to exactly one tier. Handling priority follows tier order.
- **Alternatives Considered:**
  - Keep ad-hoc classification: Harder for engineering to implement; no clear priority ordering. Rejected.
  - Michelle's 4-tier without T0: Doesn't address infrastructure failures. Extended to 5 tiers.
- **Consequences:** Clean engineering contract — every error code has a defined tier and handling rule. Cross-team communication simplified ("this is a T2 error, not a T1").

### ADR-006: Display Timing — Batched After Parallel Evaluation (UPDATED)

- **Status:** Accepted (updated from v2 progressive model)
- **Context:** v2 adopted progressive display (showing flight T3 during accom validation). With the architectural shift to parallel evaluation (ADR-008), both verticals complete before any display logic runs. Progressive display is no longer possible since there is no "during accom validation" window.
- **Decision:** Display is fully batched after parallel evaluation completes. The latency advantage of parallel evaluation (MAX vs SUM of call times) compensates for the loss of progressive display. The display sequence remains: flight T3 → opaque price → accom T3.
- **Alternatives Considered:**
  - Keep progressive display with parallel: Not possible — both calls complete concurrently, no window to show flight T3 "while accom validates."
  - Stream flight result to FE immediately: Adds complexity (WebSocket/SSE); flight terminal errors still need cancel-fast logic. Rejected for Phase 1.
- **Consequences:** Simpler FE implementation (single response with all data). User waits for MAX(flight, accom) before seeing anything, but this is faster than sequential SUM(flight, accom) in most cases.

### ADR-007: Automatic Retry for T0/T1 Errors (NEW)

- **Status:** Proposed
- **Context:** v1 brainstorming treated system errors as "cannot continue → reload." Michelle proposed automatic retry before displaying errors.
- **Decision:** Implement automatic silent retry for T0 (2 retries) and T1 (1 retry) errors before surfacing to the user. Total timeout budget: 30s.
- **Alternatives Considered:**
  - No automatic retry (user clicks "Reload"): Worse UX; unnecessary round-trip through FE. Rejected.
  - Unlimited retries: Risk of timeout and poor UX. Rejected. Cap at 2/1 respectively.
- **Consequences:** Transient failures are absorbed silently. Users only see errors when the system genuinely can't proceed.

### ADR-008: Parallel Evaluation with Sequential Display (NEW)

- **Status:** Accepted
- **Context:** ADR-001 (from v1 brainstorming) specified sequential validation — Flight completes fully before Accom begins. The rationale was "Parallel would require more complex aggregation logic and potential race conditions." In practice, the aggregation logic is identical (same tier priority rules apply regardless of call order) and no race conditions exist because the orchestrator waits for both results before processing. Meanwhile, sequential adds unnecessary latency (SUM of call times vs MAX).
- **Decision:** Evaluate both verticals in parallel. Apply a flight-priority gate after both complete. If flight is terminal (T0/T1/T2/T3-fix/T3-back), cancel the pending accom call immediately and show the flight error (cancel-fast). If flight passes gate (T3-ok/T4/SUCCESS), classify the accom result and proceed to sequential display.
- **Supersedes:** ADR-001 (Sequential Validation). All mechanics described under ADR-001 are replaced by the parallel model.
- **Alternatives Considered:**
  - Sequential (ADR-001): Proven simpler but adds unnecessary latency. ~5s happy path (sequential) vs ~3s (parallel). Worst case 60s vs 30s. Rejected.
  - Full parallel with no gate: Both results processed independently. Risks showing accom error when flight is terminal (confusing UX — "which one do I fix first?"). Rejected.
  - Parallel with wait-for-both: Always wait for both to complete before showing anything. Adds latency when flight is terminal (user waits for accom timeout unnecessarily). Rejected in favor of cancel-fast.
- **Consequences:**
  - Latency improvement: happy path MAX(2s, 3s) = 3s vs 5s; worst case 30s vs 60s.
  - Cancel-fast: flight terminal errors shown immediately (same latency as sequential).
  - Trade-off: progressive display (showing flight T3 during accom call) is no longer possible — compensated by overall faster completion. See ADR-006 update.
  - Implementation: requires async/concurrent call infrastructure in Bundling BE with cancellation support.

---

## 8. Edge Cases — PRESERVED + EXPANDED

| # | Edge Case | Tier(s) | Handling |
|---|---|---|---|
| E1 | Both flight sold out AND accom sold out | T2 + T2 | Flight T2 terminal (gate). Accom call cancelled, result discarded. |
| E2 | Flight price up + Accom price down (net zero or negative delta) | T4 + T4 | Compute net delta. If zero or negative → silent pass-through. |
| E3 | Combination error has price as ONLY change | T4 | After stripping: combination becomes empty. Suppress entire combination error. Only show bundled opaque price change. |
| E4 | Accom combination includes `paymentMethod` change | — | `paymentMethod` not relevant for bundling. Strip alongside price. |
| E5 | Accom combination includes `maxOcc` (room capacity) | — | `maxOcc` not relevant for bundling. Strip. Pass only genuinely relevant T3 changes. |
| E6 | Flight BOOKING_INTEGRATOR_FAILED (retry-able) + Accom price change | T3 + T4 | Show flight retry error first. If user retries and flight succeeds, then show accom price as opaque. |
| E7 | `changeType` field absent (old Accom API) | — | Fallback to degraded mode (ADR-004). Log for monitoring. |
| E8 | Price decreased for both flight and accom | T4 + T4 | Silent pass-through. No message. User sees lower price on payment page. |
| **E9** | **Flight returns HTTP 500** | **T0** | **Retry 2×. If still failing: show infra error. Cancel pending accom call; result discarded.** |
| **E10** | **Flight succeeds + Accom returns HTTP 500** | **T0** | **Retry accom 2×. If still failing: show accom infra error. Discard any flight T4.** |
| **E11** | **Both verticals return HTTP 500** | **T0 + T0** | **Flight T0 terminal (gate). Accom call cancelled, result discarded.** |
| **E12** | **Flight T3 (passenger validation) + Accom T4 (price change)** | **T3 + T4** | **Flight T3-fix is terminal (gate). Cancel accom. Show flight T3 (user must fix on BF). After fix and resubmit: re-run entire validation from Step 1.** |
| **E13** | **Flight T4 (price up) + Accom T2 (sold out)** | **T4 + T2** | **Show accom T2 (redirect to SRP). Discard flight T4 — flight will be re-validated after user selects new hotel.** |
| **E14** | **Accom combination with only not-relevant changes (maxOcc + paymentMethod)** | **—** | **After stripping non-relevant items: combination becomes empty. Suppress entirely. No message.** |
| **E15** | **Network timeout during accom validation (exceeds 30s budget)** | **T0** | **Abort accom call. Show timeout error. Preserve flight validation state for retry.** |

---

## 9. Implementation Effort — UPDATED

Incorporates retry logic and T0 handling as new components.

| Component | Effort | Description |
|---|---|---|
| **Error Classification Engine** (BE) | 0.5 sprint | Map every error code to tier (T0-T4), expose as Michelleal utility |
| **Retry Controller** (BE) | 0.5 sprint | Automatic retry for T0/T1 with backoff, timeout budget |
| **Error Aggregation Service** (BE) | 2 sprints | Parallel evaluation orchestrator with gate + classify + display flow |
| **Price Containment Logic** (BE) | 0.5 sprint | Intercepts T4 from both verticals, computes opaque delta |
| **Combination Error Sanitizer** (BE) | 1 sprint | Strips price from combination errors using `changeType` |
| **Action Remapper** (BE) | 0.5 sprint | Maps vertical actions to bundling navigation |
| **Bundled Price Change Message** (BE + FE) | 0.5 sprint | New opaque price change message |
| **Batched Error Display** (FE) | 1 sprint | Display all messages sequentially after parallel eval completes |

**Total Bundling team effort: ~6.5 sprints** (up from 5-6, due to retry logic and T0 handling)

**Accom team effort: ~1 sprint** (unchanged — `changeType` + `priceData`)

---

## 10. Cross-Reference

| Topic | Reference |
|---|---|
| Complete scenario permutation matrix | `deliverables/reviews/error-scenario-permutations.md` |
| Original brainstorming (v1) | `error_aggregation_brainstorm.md` |
| Michelle's proposal (matrix format) | `references/Michelle-proposal.md` |
| Compatibility review | `deliverables/reviews/error-aggregation-review.md` |
| Accommodation error spec | `references/accommodation-error-spec.tsv` |
| Flight error spec | `references/flight-error-message-error-spec.tsv` |
| Bundling initiative spec | `references/bundling_initiative_spec.md` |

<!-- CHECKPOINT id="ckpt_mo0rlbx2_9nmcjv" time="2026-04-16T00:52:11.030Z" note="auto" fixes=0 questions=0 highlights=0 sections="" -->
