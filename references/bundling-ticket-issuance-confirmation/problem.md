# Bundling Ticket Issuance Confirmation — Problem Statement

> **Status**: OPEN — Pending cross-team discussion
> **Raised by**: Order/Bundling PM
> **Affected teams**: Flight, Order/Bundling, Accommodation
> **Related spec section**: [Section 8 — Issuance, Error Handling & Order Lifecycle](../general/bundling_initiative_spec.md#8-issuance-error-handling--order-lifecycle)

---

## 1. Context: Why Sequential Issuance Exists

The bundling product (Flight + Hotel) uses **sequential issuance**: `Flight → Hotel`.

This design decision exists because **Accommodation's opaque pricing** — currently sourced from Expedia — requires the hotel booking to be invalidated if the flight cannot be issued. Expedia unlocks special rate keys (opaque bundling rates) on the condition that the hotel is purchased alongside a flight. If the flight fails, the hotel must be cancelled/refunded since the opaque price contract is void.

Sequential issuance enforces this dependency:

1. Order service requests Flight issuance first.
2. **If flight issuance fails** → Order triggers auto-refund for **both** flight and hotel items (full refund).
3. **If flight issuance succeeds** → Order proceeds to request Hotel issuance from Accommodation service.
   - **If hotel issuance fails** → Order triggers partial refund for **hotel only**. Flight remains issued.
   - **If hotel issuance succeeds** → Both items issued. Order complete.

---

## 2. The Problem: Flight's Internal Rebooking Flow

Flight service has an internal rebooking mechanism that **conflicts with the bundling issuance contract**.

### Current Flight Behavior (Non-Bundling)

When flight issuance fails (ref: step 2 above), Flight service does **not** immediately report failure. Instead, it silently attempts a rebooking flow:

1. **Rebooking is invisible to Order/Bundling service** — Flight does not notify Order or Bundling that issuance failed or that rebooking is in progress. The process is opaque until completion.
2. **If rebooking fails** → Flight finally informs Bundling service (orchestrator) that issuance has failed. At this point, the normal failure path (auto-refund both items) can proceed.
3. **If rebooking succeeds** → Flight creates a **new order ID** to replace the original, and the old order is updated to a "new booking" status.

### Why This Clashes with Bundling

There are two distinct problems when this rebooking behavior occurs within a bundling context:

#### Problem A: Order ID Replacement Breaks Bundling Structure

When Flight's rebooking succeeds (point 3 above), it generates a **new order ID** to replace the original flight order.

In the bundling data model, the flight is an **L2 order detail** under a shared **L1 bundling order ID**:

```
L1: Bundling Order ID
├── L2: Flight Order Detail ID  ← this gets replaced
└── L2: Hotel Order Detail ID
```

Flight's rebooking mechanism expects to replace the **entire order**, but in bundling, the flight is only one component of a composite order. The replacement logic cannot simply swap out the whole order — it would need to surgically replace only the flight L2 while preserving the hotel L2 and the L1 wrapper. **The current rebooking mechanism is not designed for this.**

If we force the same replacement mechanic, it would replace the entire bundling order (both flight and hotel), which is incorrect.

#### Problem B: Rebooked Flight Triggers False Issuance Failure for Hotel

Even if we could solve the order ID replacement issue, there is a second-order problem.

When Flight's rebooking is in progress, the **original flight order detail** within the bundling order would be marked as "failed issued" (or equivalent transitional state). This status propagates to the bundling orchestrator.

Per the sequential issuance contract (step 2 above), a failed flight issuance means **hotel issuance should never be triggered**. The orchestrator would interpret the original flight's failed status as a signal to abort hotel issuance and refund both items — even though Flight is still silently attempting to rebook.

This creates a race condition:
- Flight is rebooking internally (may succeed).
- Bundling orchestrator sees flight failure and starts refunding both items.
- If rebooking succeeds after refund is triggered, the system is in an inconsistent state.

---

## 3. Summary of Conflicts

| Aspect | Non-Bundling (Current) | Bundling (Required) | Conflict |
|--------|----------------------|-------------------|----------|
| Rebooking notification | Silent until complete | Needs immediate failure signal or explicit "rebooking in progress" state | Flight's opacity blocks orchestrator decision-making |
| Order ID replacement | Replace entire order | Cannot replace — flight is part of composite L1 order | Replacement mechanic incompatible with L1/L2 structure |
| Failed status impact | Triggers standalone refund | Triggers refund of **both** flight and hotel | False failure cascades to hotel, preventing issuance |
| Rebooking success handling | New order replaces old | No clear mechanism to link rebooked flight back to original bundling order | Orphaned hotel order detail; broken L1 relationship |

---

## 4. Proposed Solution (Initial Proposal, Pending Discussion)

**Differentiate the flow for bundling orders**: when the issuance request originates from a bundling context, Flight service should **skip the internal rebooking flow** and immediately report issuance failure to the Bundling orchestrator.

### Rationale

- Keeps the bundling issuance contract intact (Flight fail → refund both → clean state).
- Avoids the order ID replacement problem entirely.
- Avoids the false-failure race condition with hotel issuance.
- Minimal change surface: Flight adds a conditional check on a `is_bundling` flag (already planned for other features) to bypass rebooking.

### Trade-offs

| Pro | Con |
|-----|-----|
| Clean failure path for bundling | User loses the chance of a successful rebook on the flight side |
| No structural changes to order model | Bundling orders may have a slightly higher flight failure rate (no rebook safety net) |
| Simple implementation (flag-based branching) | Diverges Flight issuance behavior between bundling and standalone |

### Open Questions for Discussion

1. **Is the rebooking success rate high enough to justify the complexity of supporting it in bundling?** If rebooking rarely succeeds, skipping it has minimal user impact.
2. **Could Flight support a "synchronous rebooking" mode** where it attempts rebook but still returns a single success/failure response within the issuance timeout — without creating a new order ID?
3. **If we eventually want rebooking in bundling**, what would the orchestrator state machine look like? Would we need a new `REBOOKING_IN_PROGRESS` state at the L1 level?
4. **Does the rebooking mechanism create a new order in the same Order DB**, or does it go through a different path? This affects whether the L1/L2 linkage could theoretically be preserved.

---

## 5. Additional Context

### Why Opaque Pricing Matters Here

The sequential issuance requirement is **not arbitrary** — it is driven by Accommodation's opaque pricing contract:

- **Opaque pricing** means the hotel's special bundling rate is only valid when displayed as a combined price with a flight (no individual price breakdown visible to the user).
- This pricing is currently sourced from **Expedia**, which provides a new rate key with the special price as long as the price is displayed opaquely.
- If the flight cannot be issued, the hotel's opaque rate is no longer valid — Expedia's contract requires the hotel booking to be invalidated.
- This is why the bundling issuance flow **must** confirm flight success before proceeding with hotel issuance: the hotel's rate existence depends on the flight being successfully booked.

### Cascading Impact

If this problem is not resolved before launch, the following scenarios could occur in production:

1. **Stuck orders**: Flight is silently rebooking while Bundling orchestrator waits indefinitely (or times out incorrectly).
2. **Double refund**: Orchestrator refunds both items due to perceived flight failure, but Flight's rebook succeeds — resulting in a refunded hotel + issued flight under a broken order structure.
3. **Orphaned hotel bookings**: If the orchestrator proceeds with hotel issuance during the rebooking window (due to timing), and the rebook changes the flight order ID, the hotel booking is now linked to a non-existent flight order detail.

---

*This document will be updated after cross-team discussion with Flight and Bundling engineering.*

<!-- CHECKPOINT id="ckpt_mo1ahy07_dnt2j4" time="2026-04-16T09:41:25.735Z" note="auto" fixes=0 questions=0 highlights=0 sections="" -->
