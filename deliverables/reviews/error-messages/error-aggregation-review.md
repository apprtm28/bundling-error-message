# Review: Michelle Proposal vs. Error Aggregation Brainstorming

## Overall Assessment
**Readiness:** Needs Revision (Michelle Proposal); the Brainstorming document needs structural improvement
**Reviewer Confidence:** High
**Review Date:** 2026-04-16
**Documents Reviewed:**
- `error_aggregation_brainstorm.md` (Brainstorming — Checkpoint v1, 2026-04-08)
- `references/Michelle-proposal.md` (Michelle Proposal — Scenario Matrix)

---

## Scores

| Dimension | Brainstorming (1-5) | Michelle Proposal (1-5) | Notes |
|---|:---:|:---:|---|
| Clarity | 4 | 4 | Brainstorming is thorough but dense. Michelle is concise but omits rationale. |
| Specificity | 5 | 3 | Brainstorming has exact error codes, API structures, ADRs. Michelle uses abstract categories without mapping to real error codes. |
| Evidence | 5 | 2 | Brainstorming references TSVs, Confluence, API contracts. Michelle proposal has no source references. |
| Consistency | 4 | 4 | Both Michelleally consistent. Minor terminology differences between them. |
| Completeness | 4 | 2 | Brainstorming covers 12 sections + edge cases. Michelle covers only the happy-path matrix. |
| Strategic Alignment | 5 | 4 | Both align with opaque pricing requirement. Michelle doesn't address fallback or phasing. |
| **Overall** | **4.5** | **3.2** | |

---

## Compatibility Assessment

### Where They Align (Strong Foundation)

| Principle | Brainstorming | Michelle Proposal | Verdict |
|---|---|---|---|
| Sequential validation (Flight → Accom) | ✅ ADR-001 | ✅ Stated upfront | **Fully compatible** |
| Disruptive flight errors block accom validation | ✅ Rule (c), Scenarios 6-7 | ✅ "Don't do validation for Accommodation items until Flight final check is successful" | **Fully compatible** |
| Price errors suppressed and combined into opaque price | ✅ Scenarios 1-3, 5 | ✅ "Price related error will not be shown until both Flight and Accommodation final check is successful" | **Fully compatible** |
| Inventory errors redirect to Bundling SRP | ✅ Scenarios 6-7 | ✅ "Redirect user to Bundling SRP" | **Fully compatible** |
| Combination error: strip price from non-price | ✅ Scenario 5, detailed flow | ✅ "price related error should be stripped from the combination" | **Compatible on intent; Michelle lacks implementation detail** |

### Where They Diverge (Requires Reconciliation)

#### 1. Error Categorization Framework

**Brainstorming** classifies by specific error codes grouped into flight categories (SYSTEM_TECHNICAL, AIRLINE_COMMUNICATION, AVAILABILITY_SEAT_FARE, PRICE_CHANGE_TICKET, etc.) and accom error types (sold out, CP change, price change, combination, etc.). Classification is bottom-up from actual API responses.

**Michelle** introduces a clean 4-tier hierarchy:
1. Non-business related error (system/technical)
2. Inventory related error (sold out/unavailable)
3. Non-price related error (business rule changes)
4. Price related error

**Assessment**: The Michelle's framework is a **structural improvement** — it provides a cleaner mental model for engineering implementation and cross-team communication. However, the brainstorming's bottom-up mapping is essential for actual implementation. These are complementary, not conflicting.

#### 2. Non-Business Error Retry Pattern

**Brainstorming** treats SYSTEM_TECHNICAL errors as "cannot continue" → reload booking form. No retry orchestration.

**Michelle** says: "Display Flight non-business related error as-is → Do final check to Flight service again → Don't do validation for Accommodation until Flight final check is successful."

**Assessment**: The Michelle introduces an **automatic retry** concept that the brainstorming doesn't address. This is a meaningful gap in the brainstorming — should the orchestrator retry, or does the FE "Reload" button serve as the retry mechanism? The Michelle implies BE-level retry before falling through to error display, which is architecturally cleaner.

**Recommendation**: Adopt the Michelle's retry-before-display pattern but specify retry limits (max 1-2 retries with exponential backoff) and timeout constraints. The brainstorming should add this to the Error Aggregation Service specification.

#### 3. Flight Non-Price Error Display Timing

**Brainstorming** (Scenario 8): queues both flight and accom non-price errors, shows flight first then accom.

**Michelle**: "Flight non-price related error will have already been displayed before Accommodation final check is successful."

**Assessment**: Subtle but important difference. The Michelle implies flight non-price errors are shown to the user **during** the sequential validation (between flight check and accom check), not held until both validations complete. The brainstorming implies all errors are collected first, then displayed. The Michelle's approach is better for UX — users resolve flight issues immediately while accom validation runs.

**Recommendation**: Adopt the Michelle's timing model for non-price, non-disruptive flight errors. Price errors are still held until both checks complete.

#### 4. Accom Non-Business Error Handling

**Brainstorming** doesn't explicitly address accom system/technical errors as a separate category. The error taxonomy focuses on business-rule errors (the TSV categories).

**Michelle** explicitly handles: "Display Accommodation non-business related error as-is → Do final check to Accommodation service again."

**Assessment**: The brainstorming has a gap — it doesn't specify what happens when accom's `get-prebook` or `book` API returns a non-200 response or a system-level error. The Michelle correctly identifies this as a distinct category requiring retry logic.

---

## Critical Issues (Must Fix)

### CI-1: Michelle Proposal — No Implementation Path for Combination Error Stripping

The Michelle states "price related error should be stripped from the combination" but provides zero detail on HOW. The brainstorming identifies this as the hardest problem (Section 5) and proposes a specific solution (Approach E + B-lite with `changeType` field from Accom team). Without this detail, the Michelle's matrix cell is unimplementable.

**Fix**: Michelle proposal must reference the brainstorming's implementation approach or provide an alternative.

### CI-2: Michelle Proposal — No Fallback Strategy

If Accom team cannot add the `changeType` and `priceData` fields, the Michelle's proposal has no contingency. The brainstorming provides a degraded fallback (ADR-004).

**Fix**: Michelle proposal needs a fallback column in the matrix: "If combination error cannot be structurally parsed, apply degraded handling."

### CI-3: Brainstorming — Missing Error Tier Classification

The brainstorming jumps from individual error codes to scenario-level handling without an intermediate classification layer. Engineering teams need a clean mapping: `error_code → tier → handling_rule`. The Michelle's 4-tier framework fills this gap.

**Fix**: Brainstorming should adopt the 4-tier framework as the classification layer between raw error codes and orchestration logic.

### CI-4: Both Documents — No Non-200 HTTP Error Handling

Neither document addresses what happens when the vertical service itself is unreachable (network timeout, 5xx, connection refused, malformed response). The brainstorming focuses exclusively on error payloads within 200 responses. The Michelle's "non-business related error" category partially covers this but doesn't distinguish between:
- `200 + SYSTEM_TECHNICAL` error code (app-level)
- `500 Michelleal Server Error` (infra-level)
- Network timeout (connectivity-level)
- Circuit breaker open (resilience-level)

**Fix**: Both documents need explicit HTTP-level error handling added to the framework.

---

## Major Issues (Should Fix)

### MA-1: Michelle Proposal — No Action Button Remapping

The brainstorming specifies 12 action remappings (vertical actions → bundling navigation). The Michelle proposal uses generic "Redirect user to Bundling SRP" without acknowledging the underlying action mapping needed.

### MA-2: Michelle Proposal — No Edge Cases

The brainstorming identifies 8 edge cases (E1-E8). The Michelle proposal has none. Missing: price decrease silent pass-through (E8), combination error with only price change (E3), non-relevant accom errors (E4-E5).

### MA-3: Michelle Proposal — Price Decrease Handling Undefined

The brainstorming specifies price decrease = silent pass-through (E8). The Michelle's matrix doesn't distinguish between price increase and price decrease. This matters because showing "Your price has changed" for a decrease is unnecessary and creates friction.

### MA-4: Brainstorming — Scenario Coverage Gaps

The brainstorming covers 8 scenarios but misses explicit handling for:
- Flight non-business error + Accom any error (what if flight returns 500?)
- Flight retry-able error (BOOKING_INTEGRATOR_FAILED) + Accom price change (should we retry flight, then still show accom price change?)
- Accom system error (non-200) in any combination

### MA-5: Michelle Proposal — No Analytics/Tracker Impact

The brainstorming identifies tracker data modification needs (eventDescription stripping). The Michelle doesn't address analytics instrumentation at all.

---

## Minor Issues (Nice to Have)

### MI-1: Terminology Alignment
- Brainstorming uses "Disruptive" / "Non-disruptive" / "May disrupt"
- Michelle uses "Non-business" / "Inventory" / "Non-price" / "Price"
- These are complementary but the combined document should use a unified taxonomy

### MI-2: Michelle Proposal Formatting
- The Markdown table is hard to parse due to `<br>` tags in cells
- Would benefit from a cleaner rendering format

---

## Strengths

### Brainstorming
- Exceptional depth on the Combination Error Loophole (Sections 4-5)
- Concrete API response structures with examples
- ADRs provide decision traceability
- Edge case coverage is thorough
- Fallback strategy (ADR-004) shows mature contingency thinking
- Accom cooperation request template (Appendix B) is immediately actionable

### Michelle Proposal
- Clean 4-tier categorization framework is immediately useful for engineering
- Cross-vertical matrix format makes every permutation explicit
- The "successful = no non-business error AND inventory still available" definition is precise and implementable
- Display sequencing rule ("Flight non-price errors shown before accom check completes") is a UX improvement
- Retry pattern for non-business errors adds resilience the brainstorming lacks

---

## Recommendation

**Direction: Improve the initial brainstorming** by synthesizing the Michelle's structural improvements into a new version, rather than trying to backfill the Michelle's proposal with the brainstorming's depth.

**Rationale:**
1. The brainstorming contains irreplaceable implementation details (API structures, Approach E + B-lite, ADRs, edge cases, fallback strategy)
2. The Michelle's contribution is a **structural framework** that organizes the brainstorming's content more clearly
3. Building up from the brainstorming (adding the Michelle's framework) is less work than building up from the Michelle's proposal (adding all implementation details)

**Specific synthesis actions:**
1. Add the Michelle's 4-tier error classification as a new Section 3.3 "Error Classification Tiers"
2. Map every error code from the TSVs into the 4 tiers (this becomes the canonical mapping)
3. Replace the brainstorming's 8 scenarios with the Michelle's full cross-product matrix (expanded with implementation details)
4. Add non-200 HTTP error handling as a 5th error tier
5. Adopt the Michelle's display timing model (flight non-price shown during validation, not held)
6. Add retry semantics for non-business errors from the Michelle's approach
7. Preserve all ADRs, edge cases, and implementation details from the brainstorming
8. Add the complete scenario permutation matrix as the definitive reference (Task 2)

**Deliverable**: `deliverables/reviews/error-handling-improved-recommendation.md`
