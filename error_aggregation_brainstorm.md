# Bundling Error Aggregation — Brainstorming & Design

> **Status**: BRAINSTORMING — Checkpoint v1
> **Problem Owner**: Agung Putra Pratama (Order PM)
> **Date**: 2026-04-08
> **Agents**: PM Researcher + Engineering Architect

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Architecture Context](#2-architecture-context)
3. [Error Taxonomy — Both Verticals](#3-error-taxonomy--both-verticals)
4. [The Core Tension](#4-the-core-tension)
5. [The Combination Error Loophole](#5-the-combination-error-loophole)
6. [Open Questions](#6-open-questions)
7. [Solution Approaches](#7-solution-approaches)
8. [Decision Matrix](#8-decision-matrix)
9. [Recommended Approach](#9-recommended-approach)
10. [ADRs](#10-adrs)

---

## 1. Problem Statement

### 1.1 HARD REQUIREMENT (Non-Negotiable)

**Opaque pricing must be maintained at all times.** The bundled price is displayed as:

```
Per-pax bundled price = (Flight total + Hotel total) / Number of travelers
```

Individual flight or accommodation prices must **NEVER** be shown to users. Violating this breaks partnership agreements (opaque bundling rates) and causes catastrophic integration impact.

### 1.2 What We're Trying To Do

In the bundling flow (Flight + Hotel), both verticals return their own error messages during validation (Booking Form → Payment Page). We need to aggregate these errors into a unified experience **without reinventing the wheel** — reusing each vertical's existing error handling as much as possible.

### 1.3 The Initial Plan (Confirmed)

| # | Rule | Rationale |
|---|------|-----------|
| a | Flight error messages show **first** | Flight validates first in sequential flow |
| b | Accom error messages show **afterwards** | After flight validation passes or has non-blocking errors |
| c | If flight error has **backward-only action** (e.g., sold out → go to SRP), do NOT proceed to show accom errors | No point validating hotel if flight is dead |
| d | If **price-related** messages from either vertical, **do NOT show them as-is** — combine into a single opaque bundled price change message | Protects opaque pricing |

### 1.4 The Problem With Rule (d) — The Loophole

Accommodation has a **"Combination Error"** type that bundles multiple rate plan changes into a single error popup. This combined error can include:
- Price changes
- Cancellation policy changes
- Room capacity changes
- Meal plan changes
- Value-added benefit changes
- Travel policy changes

The combined error popup has a **"See changes & continue booking"** button (actionId: `OPEN_RATE_PLAN_CHANGES_PREBOOK` or `OPEN_RATE_PLAN_CHANGES_BOOK`) that opens a bottom sheet showing **per-room details including individual room prices**.

**The problem**: You can't simply blacklist or suppress the price-related error because it's **interleaved with non-price changes** in a single response. Options and their downsides:

| Option | Downside |
|--------|----------|
| Block the entire combined error | Users lose visibility into critical non-price changes (cancellation policy becoming non-refundable, meal plan changes, etc.) |
| Hide the "See changes" button, show "Continue booking" only | Same as above — bottom sheet with detail is gone, non-price changes are invisible |
| Let the combined error through as-is | Individual accommodation prices leak through the bottom sheet, breaking opaque pricing |

**Additional complication**: Accommodation has **aggressive caching**. When users click "See other rooms" to go back, the room list still shows old (cached) data. The Accommodation team *relies on* the error message to communicate changes to users because the room list won't reflect reality immediately.

**Another complication**: Even if we skip the bottom sheet, the payment page's order summary does **NOT** show all details that may have changed (e.g., cancellation policy is NOT shown on payment page). So users proceed to payment without knowing their cancellation policy changed from "100% Refund & Reschedule" to "Non-refundable."

---

## 2. Architecture Context

### 2.1 Bundling Validation Flow (from bundling_initiative_spec.md §8.3)

```
Booking Form → [User clicks "Continue to Payment"]
  → Bundling BE calls Flight service for validation
  → Bundling BE calls Accom service for validation
  → Bundling BE aggregates results
  → Bundling FE renders error(s) or proceeds to Payment Page
```

The Bundling/Order service acts as the **orchestrator** — it sits between the FE and both verticals.

### 2.2 Accommodation Error Response Structure

Source: [TACM] Error Code Mapping & Handling - Booking Form (Confluence)

```
GET/POST → tix-hotel-search/hotel/v5/get-prebook
GET/POST → tix-hotel-search/hotel/v2/book

Response (error case):
{
  "code": "BUSINESS_ERROR",
  "errorData": {
    "errorCode": "ROOM_UNAVAILABLE" | "PRICE_CHANGE_*" | etc.,
    "uiType": "BOTTOMSHEET" | "SNACKBAR" | "BANNER",
    "error": {
      "imageUrl": "...",
      "message": { "title": "...", "description": "..." },
      "trackerData": { ... },
      "actions": [
        { "text": "...", "value": "NAVIGATE_PREV_PAGE|RETRY|...", "style": "PRIMARY|SECONDARY", ... }
      ]
    }
  }
}
```

**For Combination Errors** (rate plan changes), additional field:
```
"newRatePlanChanges": {
  "banner": { "title": "...", "body": "...", "action": { "actionId": "OPEN_RATE_PLAN_CHANGES_PREBOOK" } },
  "popup": {
    "details": [
      {
        "roomName": "Sawangan Junior Deluxe",
        "details": [
          { "text": "Price: From IDR 1,300,000 to IDR 1,350,000", "subText": "" },
          { "text": "Cancellation policy: Non-refundable", "subText": "" },
          { "text": "Room capacity: 2 adults, 1 child", "subText": "" },
          { "text": "Meal: breakfast (2 pax)", "subText": "" }
        ]
      }
    ],
    "primary": { "title": "Continue booking", "actionId": "NONE" },
    "secondary": { "title": "See other rooms", "actionId": "NAVIGATE_PREV_PAGE" }
  }
}
```

**Critical observation**: The `newRatePlanChanges.popup.details[].details[]` is **structured data** — each change is a separate object with `text` and `subText`. Price items are identifiable (text starts with "Price:"). This means **programmatic filtering is feasible**.

### 2.3 Flight Error Response Structure

Source: flight-error-message-error-spec.tsv

Flight errors follow a different contract:
- Category-based classification (PRICE_CHANGE_TICKET, AVAILABILITY_SEAT_FARE, AIRLINE_COMMUNICATION, etc.)
- Error codes like `PRICE_CHANGE_ADJUSTMENT_UP`, `PRICE_CHANGE_AIRLINES_DOWN`, etc.
- Message templates with `%s` placeholders for prices: "The total payment has changed from %s to %s"
- Actions: button name, style (PRIMARY/SECONDARY), action destination (paymentPage, searchResultPage, bookingFormPage)

**Key difference from Accom**: Flight price messages embed old/new prices directly in the message body text using `%s` substitution. There's no separate structured data for the price values.

### 2.4 Accom Error Types and Bundling Relevance

From accommodation-error-spec.tsv, each error type has a "Compatibility with Flight" column:

| Error Type | Compatibility | Bundling Treatment |
|-----------|--------------|-------------------|
| All rooms sold out | Relevant; Separated | Show as-is (no price leak) |
| Room capacity change | Not Relevant | Skip in bundling (pax managed by bundling) |
| Cancellation policy flag change | Relevant; Separated | Show as-is (no price leak) |
| Cancellation policy detail change | Relevant; Separated | Show as-is (no price leak) |
| Travel policy change | Not Relevant | Skip (B2B only) |
| Meal plan change | Relevant; Separated | Show as-is (no price leak) |
| Value-added benefit change | Relevant; Separated | Show as-is (no price leak) |
| Payment method change | Not Relevant | Skip (PAH excluded from bundling) |
| **Price change (standalone)** | **Relevant; Combined** | **Must combine with flight into opaque price** |
| Insurance error | Relevant; Separated | Show as-is |
| **Combination error (multi-change)** | **Not Relevant** | **THE LOOPHOLE — contains price mixed with non-price changes** |

**Key insight**: When a single change happens (e.g., only cancellation policy changed), Accommodation sends a **standalone error** that's cleanly separated. The problem ONLY occurs when **multiple changes happen simultaneously** and Accommodation bundles them into a single "Combination Error."

---

## 3. Error Taxonomy — Both Verticals

### 3.1 Flight Errors Relevant to Bundling Phase 1

From flight-error-message-error-spec.tsv (filtered by "Shown to Users in 1st phase = Yes"):

| Category | Error Code | Impact | Bundling Concern |
|----------|-----------|--------|-----------------|
| ADDONS_BAGGAGE_CHANGE | INCLUSIVE_BAGGAGE_CHANGE | Non-disruptive | None — no price leak |
| AIRLINE_COMMUNICATION | FLIGHT_CART_FAILED | Disruptive (→ SRP) | Blocks accom validation (Rule c) |
| AIRLINE_COMMUNICATION | BOOKING_INTEGRATOR_FAILED | May disrupt | Retry or → SRP |
| AUTH_LOGIN | B2B_LOGIN_NEEDED | May disrupt | N/A for B2C Phase 1 |
| AVAILABILITY_SEAT_FARE | NO_SEAT / BOOKING_INTEGRATOR_NO_SEAT | Disruptive (→ SRP) | Blocks accom validation (Rule c) |
| AVAILABILITY_SEAT_FARE | FARE_ALREADY_SOLD_OUT | Disruptive (→ PRP/SRP) | Blocks accom validation (Rule c) |
| BOOKING_DOUBLE_LIMIT | DOUBLE_BOOKING_ERROR | May disrupt | Show as-is |
| BOOKING_SESSION_CART | CART_NOT_EXIST | Cannot continue | Blocks everything |
| PASSENGER_DATA_VALIDATION | Various | Non-disruptive to cannot continue | Show as-is |
| **PRICE_CHANGE_TICKET** | **PRICE_CHANGE_ADJUSTMENT_UP/DOWN** | **Non-disruptive** | **PRICE LEAK — must intercept** |
| **PRICE_CHANGE_TICKET** | **PRICE_CHANGE_AIRLINES_UP/DOWN** | **Non-disruptive** | **PRICE LEAK — must intercept** |
| **PRICE_CHANGE_TICKET** | **PRICE_CHANGE_GENERAL_UP/DOWN** | **Non-disruptive** | **PRICE LEAK — must intercept** |
| QUEUE_TIMEOUT | BOOKING_FLOODING_VALIDATION | Cannot continue | Show as-is |
| SYSTEM_TECHNICAL | Various | Cannot continue | Show as-is |

### 3.2 Accom Errors Relevant to Bundling

| Error Type | When | Impact | Bundling Concern |
|-----------|------|--------|-----------------|
| All rooms sold out | Prebook / Book | Disruptive | No price leak; redirect to bundling SRP |
| Cancellation policy flag | Prebook / Book | May disrupt | No price leak; show as-is |
| Cancellation policy detail | Prebook / Book | Non-disruptive | No price leak; show as-is |
| Meal plan/pax | Prebook / Book | Non-disruptive | No price leak; show as-is |
| Value added | Prebook / Book | Non-disruptive | No price leak; show as-is |
| Insurance error | Prebook / Book | Non-disruptive | No price leak; show as-is |
| **Price change** | **Prebook / Book** | **Non-disruptive** | **PRICE LEAK — must intercept** |
| **Combination** | **Prebook / Book** | **May disrupt** | **PRICE LEAK via bottom sheet details** |

---

## 4. The Core Tension

```
     OPAQUE PRICING PROTECTION              USER TRANSPARENCY
     ─────────────────────────              ──────────────────
     Must hide individual prices      vs.   Must show non-price changes
     from flight & accom                    (cancellation policy, meals, etc.)
                                            that affect user's decision

     The Combination Error is the bridge that connects both concerns.
     It bundles price + non-price changes into ONE response.
```

### 4.1 Why This Is Hard

1. **Accommodation's Combination Error is atomic** — It's designed as a single unit where all changes are presented together. The "See changes & continue booking" button opens a bottom sheet that shows ALL changes per room, including price.

2. **You can't split the response easily from FE alone** — The `popup.body` (HTML) and `newRatePlanChanges.popup.details` are coupled. If you hide the button that opens the bottom sheet, you lose the entire detail view.

3. **Going back doesn't help** — Accommodation has aggressive caching. Room list shows stale data. The error message IS the mechanism for communicating changes.

4. **Payment page doesn't fill the gap** — Even if users proceed to payment, the payment page order summary doesn't show cancellation policy or meal plan changes. Users would proceed unaware.

---

## 5. The Combination Error Loophole — Deep Dive

### 5.1 Anatomy of the Combination Error Response

When Accommodation detects multiple changes between room list selection and prebook/book, it returns:

**popup object** (the initial bottom sheet):
```
{
  "type": "BOTTOMSHEET",
  "title": "{X} details have changed",
  "body": "The price, cancellation policy, room capacity, and meals have been updated.",
  "action": {
    "primary": { "title": "See changes & continue booking", "actionId": "OPEN_RATE_PLAN_CHANGES_PREBOOK" },
    "secondary": { "title": "See other rooms", "actionId": "NAVIGATE_PREV_PAGE" }
  }
}
```

**newRatePlanChanges.popup** (the detail bottom sheet opened by "See changes"):
```
{
  "details": [
    {
      "roomName": "Sawangan Junior Deluxe",
      "details": [
        { "text": "Price: From IDR 1,300,000 to IDR 1,350,000" },     ← PRICE LEAK
        { "text": "Cancellation policy: Non-refundable" },              ← Important for user
        { "text": "Room capacity: 2 adults, 1 child" },                 ← Important for user
        { "text": "Meal: breakfast (2 pax)" }                           ← Important for user
      ]
    }
  ]
}
```

### 5.2 The Leakage Vectors

| Vector | What Leaks | Where |
|--------|-----------|-------|
| `popup.body` text | Lists "price" as one of the changed items (not the actual amount) | Initial bottom sheet |
| `newRatePlanChanges.popup.details[].details[]` | Actual IDR amounts per room | Detail bottom sheet |
| `newRatePlanChanges.banner.body` | May reference price changes | Booking form banner |

### 5.3 What the Current Plan Does vs. What's Needed

| Current Plan | What Happens | Gap |
|-------------|-------------|-----|
| Hide "See changes" button, show "Continue booking" only | Bottom sheet never opens, no price leak | Non-price changes (cancellation policy, meals) become invisible |
| User proceeds to payment page | Payment page shows order summary | Payment page does NOT show cancellation policy, meal plan, or other changed details |
| User clicks "See other rooms" to go back | Room list loads | Room list shows **stale cached data** — same rooms, same old details |

**Result**: User proceeds to payment and books a room without knowing their cancellation policy changed from "100% Refund & Reschedule" to "Non-refundable." This is a UX failure and potential legal/trust issue.

---

## 6. Open Questions — RESOLVED

| # | Question | Answer |
|---|----------|--------|
| Q1 | Does Bundling BE proxy all API calls? | **Hybrid** — Bundling BE handles BF→Payment validation (prebook/book); FE calls verticals directly for search, room list, etc. |
| Q2 | Validation timing at BF→Payment? | **Sequential** — Flight validates first. If flight price change, collect new price temporarily. Then accom validates. If accom price change, collect it. **Aggregate both before showing any error to user.** |
| Q3 | Can Bundling BE rewrite `newRatePlanChanges`? | **Feasible but fragile** — `text` field is NOT reliably identifiable by prefix (see Q5). Need Accom team cooperation for a type identifier. |
| Q4 | Accom team willing to change? | **Maybe, but only minimal changes.** Won't redesign their error system; may add small fields or flags. |
| Q5 | Price identifiable in `text` field? | **Can vary** — "Price:", "Harga:" (Indonesian), other formats. **Text-based parsing is unreliable.** |
| Q6 | Flight error JSON structure? | **Different but structured.** MongoDB document: `errorCode`, `priority`, `imageUrl`, `buttons[]` (each with `text.{lang}`, `style`, `type`, `eventLabel`), `message.{lang}.title`, `message.{lang}.subTitle`. Price messages use `%s` placeholders filled before delivery to FE. |
| Q7 | Can `popup.body` be rewritten? | **Yes if Bundling BE rewrites.** But body text varies by language — fragile. Better: get Accom to generate body without "price" when type identifier is available. |
| Q8 | Tracker implications? | **Yes** — Amplitude tracks `eventDescription` like `prebook;rateSpecChanges:price,cpFlag,cpDetail`. If price is stripped, tracker description must also update. Tracker is client-side; Bundling BE should modify tracker data before forwarding. |

### Q2 Deep Dive: The Sequential Aggregation Flow

This is a critical architectural decision that directly shapes the solution:

```
User clicks "Continue to Payment" on Booking Form
  │
  ▼
┌─────────────────────────────────────────────────────┐
│ Bundling BE: Validation Orchestration               │
│                                                     │
│ Step 1: Call Flight validation                      │
│   ├── If DISRUPTIVE error (sold out, system error)  │
│   │   → STOP. Return flight error only.             │
│   │     Rule (c): Don't even check accom.           │
│   │                                                 │
│   ├── If PRICE CHANGE                               │
│   │   → Save: flight_old_price, flight_new_price    │
│   │     Suppress flight price error. Continue.      │
│   │                                                 │
│   └── If OTHER non-disruptive error                 │
│       → Queue for display. Continue.                │
│                                                     │
│ Step 2: Call Accom validation                       │
│   ├── If DISRUPTIVE error (all rooms sold out)      │
│   │   → Return accom error (redirect to SRP)        │
│   │                                                 │
│   ├── If STANDALONE PRICE CHANGE                    │
│   │   → Save: accom_old_price, accom_new_price      │
│   │     Suppress accom price error. Continue.       │
│   │                                                 │
│   ├── If COMBINATION ERROR (price + non-price)      │
│   │   → THE HARD CASE (see Section 7)               │
│   │                                                 │
│   └── If NON-PRICE error only                       │
│       → Queue for display. Continue.                │
│                                                     │
│ Step 3: Aggregate & Return                          │
│   ├── If any price changed:                         │
│   │   → Compute opaque delta:                       │
│   │     old_opaque = (flight_old + accom_old) / pax │
│   │     new_opaque = (flight_new + accom_new) / pax │
│   │   → Emit bundling price change message          │
│   │                                                 │
│   ├── Return queued non-price errors                │
│   │   (flight errors first, then accom errors)      │
│   │                                                 │
│   └── FE shows errors sequentially to user          │
└─────────────────────────────────────────────────────┘
```

---

## 7. Solution Approaches — Evaluated

### 7.1 Why Pure Text Parsing Is Off The Table

Since price text varies by language ("Price:", "Harga:", potentially other formats), **any approach that relies on parsing the `text` field to identify price items is unreliable**. This eliminates pure Approach A (BE filtering without Accom cooperation) and Approach C (client-side filtering).

We need either:
1. A **type identifier** from Accom to distinguish price items from non-price items in the details array
2. OR a completely different mechanism to handle the combination error

### 7.2 The Minimal Ask From Accom Team

Given Accom will only make **minimal changes**, the smallest useful thing to request is:

**Add a `changeType` field to each item in `newRatePlanChanges.popup.details[].details[]`:**

```json
{
  "roomName": "Sawangan Junior Deluxe",
  "details": [
    { "text": "Price: From IDR 1,300,000 to IDR 1,350,000", "subText": "", "changeType": "price" },
    { "text": "Cancellation policy: Non-refundable", "subText": "", "changeType": "cpFlag" },
    { "text": "Room capacity: 2 adults, 1 child", "subText": "", "changeType": "maxOcc" },
    { "text": "Meal: breakfast (2 pax)", "subText": "", "changeType": "mealPax" }
  ]
}
```

**Why this is minimal**: Accom already categorizes these changes internally — the `eventDescription` tracker field contains `rateSpecChanges:price,cpFlag,cpDetail,maxOcc,mealPax,mealFlag,valueAdded`. They already know the type; they just don't expose it in the details array.

**Additionally**, request Accom to return the **raw numeric price values** (old and new total) in a separate field:

```json
"newRatePlanChanges": {
  ...,
  "priceData": {
    "oldTotalAAT": 1300000,
    "newTotalAAT": 1350000,
    "currency": "IDR"
  }
}
```

This enables Bundling BE to compute the opaque price delta without parsing text.

### 7.3 Approach Evaluation (Given Constraints)

#### Approach A: Pure BE Rewriting (Without Accom Help) — ELIMINATED

Unreliable due to language-variant text parsing. Too fragile.

#### Approach B: Full Accom Bundling Flag — OVERSCOPED

Requires Accom to redesign error responses. Won't happen.

#### Approach C: Client-Side Filtering — ELIMINATED

Same text parsing problem. Worse: tripled effort across 3 platforms. Opaque price enforcement at client level is unsafe.

#### Approach D: BE Classifies, FE Renders — PARTIAL FIT

Better than C but still requires FE to strip price. Classification is useful but doesn't solve the rendering problem alone.

#### Approach E: Opaque Price Summary + Filtered Pass-Through — BEST FIT

**Recommended: Approach E with Minimal Accom Cooperation (Approach B-lite)**

### 7.4 Recommended Approach: E + B-lite — "Filter & Wrap"

#### The Two-Layer Strategy

**Layer 1: Price Containment (Bundling BE)**
- Intercept all price-related errors from both verticals
- Compute combined opaque price change
- Emit a single bundling-level price change message

**Layer 2: Non-Price Pass-Through (Bundling BE + FE)**
- For accom combination errors: strip price items using `changeType` field (from Accom B-lite cooperation)
- For non-price accom errors: pass through as-is
- For non-price flight errors: pass through with action remapping

#### Detailed Error Flow by Scenario

**Scenario 1: Flight price change only**
```
Flight: PRICE_CHANGE_GENERAL_UP (from %s to %s)
Accom: SUCCESS

→ Bundling BE: Suppress flight price error
→ Bundling BE: Compute opaque delta using flight old/new totals
→ Emit: "Your bundled price has been updated from IDR X/pax to IDR Y/pax"
→ Button: "OK, Got It" → paymentPage
```

**Scenario 2: Accom price change only**
```
Flight: SUCCESS
Accom: PRICE_CHANGE (standalone, not combination)

→ Bundling BE: Suppress accom price error
→ Bundling BE: Compute opaque delta using accom old/new totals
→ Emit: "Your bundled price has been updated from IDR X/pax to IDR Y/pax"
→ Button: "OK, Got It" → paymentPage
```

**Scenario 3: Both flight and accom price change**
```
Flight: PRICE_CHANGE_AIRLINES_UP
Accom: PRICE_CHANGE_ALL

→ Bundling BE: Suppress both price errors
→ Bundling BE: Compute combined opaque delta
→ Emit: Single bundled price change message with combined delta
→ Button: "OK, Got It" → paymentPage
```

**Scenario 4: Accom combination error WITHOUT price (e.g., cancellation + meal)**
```
Flight: SUCCESS
Accom: Combination (cpFlag + mealPax, NO price change)

→ Bundling BE: Pass through accom combination error as-is
→ No price to strip — no modification needed
→ FE renders accom's native combination error bottom sheet
```

**Scenario 5: Accom combination error WITH price (THE HARD CASE)**
```
Flight: SUCCESS or PRICE_CHANGE
Accom: Combination (price + cpFlag + mealPax)

→ Bundling BE:
  1. Extract accom price data from `priceData` field (B-lite cooperation)
  2. Strip price items from `newRatePlanChanges.popup.details[].details[]`
     using `changeType === "price"` (B-lite cooperation)
  3. Rewrite `popup.body` to exclude "price" from changed items list
     (e.g., "The cancellation policy and meals have been updated")
  4. Rewrite `popup.action.primary` from "See changes & continue booking"
     to keep it (since bottom sheet now only shows non-price changes)
  5. Update `tracker.eventDescription` to remove "price" from rateSpecChanges
  6. If flight also had price change, combine both price deltas
  7. Compute opaque price delta and emit separate bundling price message

→ FE receives TWO messages in sequence:
  1. Bundling opaque price change: "Your bundled price updated from IDR X to IDR Y/pax"
     → Button: "OK, Got It"
  2. Accom non-price changes: "The cancellation policy and meals have been updated"
     → Button: "See changes & continue booking" (opens bottom sheet with ONLY
       non-price changes — safe, no price leak)
```

**Scenario 6: Flight sold out**
```
Flight: NO_SEAT / FLIGHT_CART_FAILED

→ Bundling BE: Return flight error immediately (Rule c)
→ DO NOT proceed to accom validation
→ FE shows: "Tickets sold out. Find another flight."
→ Button: Redirect to Bundling SRP
```

**Scenario 7: Accom sold out (all rooms)**
```
Flight: SUCCESS
Accom: ROOM_UNAVAILABLE

→ Bundling BE: Pass through accom error with remapped action
→ FE shows: "All rooms are fully booked"
→ Button: Redirect to Bundling SRP (NOT accom room list directly)
```

**Scenario 8: Flight non-price error + Accom non-price error**
```
Flight: INCLUSIVE_BAGGAGE_CHANGE
Accom: cpFlag change (standalone)

→ Bundling BE: Queue both, flight first
→ FE shows flight error first: "Baggage allowance changed" → "OK, Got It"
→ FE shows accom error second: "Cancellation policy changed" → "Continue booking"
```

#### Action Button Remapping

Flight and Accom error actions reference vertical-specific pages. In bundling, these need remapping:

| Original Action | Bundling Remap |
|----------------|---------------|
| Flight: `searchResultPage` | → Bundling SRP |
| Flight: `productReviewPage` | → Bundling SRP (with "Change flight" auto-triggered) |
| Flight: `bookingFormPage` | → Bundling Booking Form (stay) |
| Flight: `paymentPage` | → Bundling Payment Page |
| Accom: `NAVIGATE_PREV_PAGE` | → Bundling SRP (with hotel section, flight preserved) |
| Accom: `CHANGE_DATES` | → Bundling SRP (with date picker open) |
| Accom: `NAVIGATE_PAYMENT_PAGE` | → Bundling Payment Page |
| Accom: `REFRESH` | → Refresh Bundling Booking Form |
| Accom: `NONE` | → Dismiss dialog |
| Accom: `OPEN_RATE_PLAN_CHANGES_PREBOOK` | → Open filtered bottom sheet (price stripped) |
| Accom: `OPEN_RATE_PLAN_CHANGES_BOOK` | → Open filtered bottom sheet (price stripped) |

---

## 8. Decision Matrix

| Criterion | Weight | E + B-lite (Recommended) | Pure BE Rewrite (A) | Full Accom Flag (B) | Client-Side (C) |
|-----------|--------|:------------------------:|:-------------------:|:-------------------:|:---------------:|
| Opaque price protection | MUST | **YES** | Fragile (text parsing) | YES | Fragile |
| Preserves non-price visibility | HIGH | **YES** (full bottom sheet for non-price) | Risky (may break) | YES | Risky |
| Minimal reinvention | HIGH | **HIGH** (reuses accom UI, only strips price) | Medium | Low (high accom effort) | Low |
| Cross-team dependency | MEDIUM | **Low** (only `changeType` + `priceData` from Accom) | None | High | None |
| Implementation speed | MEDIUM | **2-3 sprints** (Bundling BE + minimal Accom) | 1-2 sprints but fragile | 4+ sprints | 3+ sprints × 3 platforms |
| Resilience to accom API changes | MEDIUM | **Good** (type-based, not text-based) | Poor | Good | Poor |
| Platform consistency (3 platforms) | MEDIUM | **Excellent** (BE handles filtering; FE is thin) | Good | Excellent | Poor |
| UX quality | HIGH | **Best** (users see opaque price + all non-price detail) | Good if it works | Best | Fragile |

---

## 9. Recommended Approach: E + B-lite ("Filter & Wrap")

### 9.1 Summary

Use the Bundling BE as an **error aggregation and sanitization proxy** at the BF→Payment validation step. Request **two minimal fields** from Accom team (`changeType` per detail item, `priceData` with raw numbers). Bundling BE strips price from combination errors, computes opaque price delta, and returns a clean error sequence to FE.

### 9.2 What Bundling Team Builds

| Component | Effort | Description |
|-----------|--------|-------------|
| **Error Aggregation Service** (BE) | 2 sprints | Sequential validation orchestrator that classifies, filters, and remaps errors from both verticals |
| **Price Containment Logic** (BE) | 0.5 sprint | Intercepts PRICE_CHANGE from flight (by `errorCode`) and accom (by `changeType`), computes opaque delta |
| **Combination Error Sanitizer** (BE) | 1 sprint | Strips price items from `newRatePlanChanges.popup.details`, rewrites `popup.body` and `tracker.eventDescription` |
| **Action Remapper** (BE) | 0.5 sprint | Maps vertical-specific actions to bundling navigation targets |
| **Bundled Price Change Message** (BE + FE) | 0.5 sprint | New error message type: opaque price change with bundling-specific illustration and copy |
| **Error Display Sequence** (FE) | 1 sprint | Renders error queue sequentially: flight non-price → bundled price → accom non-price |

**Total Bundling team effort: ~5-6 sprints**

### 9.3 What Accom Team Provides (Minimal Ask)

| Change | Effort | Description |
|--------|--------|-------------|
| Add `changeType` to `newRatePlanChanges.popup.details[].details[]` | 0.5 sprint | Add field they already track internally in `eventDescription`. Values: `price`, `cpFlag`, `cpDetail`, `maxOcc`, `travelPolicy`, `mealPax`, `mealFlag`, `valueAdded`, `paymentMethod` |
| Add `priceData` object to `newRatePlanChanges` | 0.5 sprint | Return `{ oldTotalAAT, newTotalAAT, currency }` when price changes are detected. Already computed internally for text generation. |

**Total Accom team effort: ~1 sprint**

### 9.4 What We Explicitly Do NOT Build

- New error message system (we reuse both verticals' existing messages)
- New bottom sheet UI (we reuse accom's existing "Rooms with changes" bottom sheet, just with price stripped)
- Custom error illustrations (we reuse existing illustrations from both verticals)
- New analytics events (we modify existing tracker data to reflect filtered state)

### 9.5 Fallback If Accom Team Declines

If Accom team cannot add `changeType` and `priceData`, the fallback is:

**Phase 1 Degraded Mode:**
1. For combination errors: **hide the "See changes" button entirely** (the current plan from problem_statement.txt)
2. Accept the UX gap: users won't see non-price changes in the bottom sheet
3. Add a **booking form banner** (like accom's existing `newRatePlanChanges.banner`) that lists which non-price components changed WITHOUT detail — e.g., "Your room's cancellation policy and meal plan have been updated. Review details before proceeding."
4. This banner can be generated by Bundling BE from the `tracker.eventDescription` field which lists change types as `rateSpecChanges:price,cpFlag,mealPax`

**Why this fallback works partially:**
- The `eventDescription` field reliably contains the change type identifiers (not language-dependent)
- We can extract `cpFlag`, `mealPax`, etc. from this comma-separated list
- We can generate a generic "these things changed" banner without showing specific values (which would require text parsing)
- Users at least KNOW something changed, even if they can't see the exact old→new values
- Opaque pricing is protected because we never show any price

**What this fallback sacrifices:**
- Users can't see *what exactly* changed in cancellation policy (e.g., from refundable to non-refundable)
- Users can't see *what exactly* changed in meal plan (e.g., breakfast to lunch)
- Accommodation's aggressive caching means going back to room list shows stale data — so there's no way for users to see the new reality

---

## 10. ADRs (Architecture Decision Records)

### ADR-001: Sequential Validation with Price Aggregation

- **Status:** Proposed
- **Context:** Bundling flow validates at BF→Payment with both Flight and Accom services. Both can independently detect price changes. Individual prices must never be shown (opaque pricing).
- **Decision:** Validate flight first, then accom sequentially. Collect price deltas from both. Suppress individual price change errors. Compute combined opaque price delta. Show single bundled price message.
- **Alternatives Considered:**
  - Parallel validation: Would require more complex aggregation logic and potential race conditions. Rejected for Phase 1 simplicity.
  - Show vertical price changes separately: Violates opaque pricing requirement. Rejected as non-viable.
- **Consequences:** Slightly higher latency (sequential, not parallel). Simpler aggregation logic. Guaranteed opaque pricing.

### ADR-002: Combination Error Sanitization via Type Identifier

- **Status:** Proposed (depends on Accom team cooperation)
- **Context:** Accom's combination error bundles price with non-price changes in a single response. The `text` field format varies by language, making text-based parsing unreliable.
- **Decision:** Request Accom team to add `changeType` field to each detail item. Bundling BE filters items by type, strips price items, rewrites body text and tracker data.
- **Alternatives Considered:**
  - Text-based regex parsing: Unreliable across languages. Rejected.
  - Hide entire combination error: Loses critical non-price change visibility. Accepted only as fallback.
  - Ask Accom to split combination error into two responses: Too large an ask for Accom team. Rejected.
- **Consequences:** Minimal Accom team effort (~1 sprint). Reliable type-based filtering. Preserves full non-price change UX. Enables bottom sheet to show non-price changes without price leak.

### ADR-003: Action Button Remapping at Orchestrator Level

- **Status:** Proposed
- **Context:** Flight and Accom error actions reference vertical-specific pages (Flight SRP, Room List, etc.) that don't exist in the bundling flow.
- **Decision:** Bundling BE rewrites action targets before forwarding to FE. Mapping table maintained in syspar for configurability.
- **Alternatives Considered:**
  - FE-side remapping: Would require each platform to maintain mappings independently. Rejected for consistency.
  - New error actions: Would require vertical teams to add bundling-specific actions. Overscoped.
- **Consequences:** FE receives bundling-native action targets. Action mapping is centralized and configurable. New vertical error types may need mapping updates.

### ADR-004: Fallback Strategy Without Accom Cooperation

- **Status:** Proposed (contingency)
- **Context:** If Accom team cannot add `changeType` and `priceData` fields before Phase 1 launch.
- **Decision:** Fallback to degraded mode: suppress combination error "See changes" button, show generic banner listing changed component names (from `eventDescription` tracker field), accept UX gap where users can't see specific change values.
- **Alternatives Considered:**
  - Delay launch until Accom cooperates: Unacceptable for Q2 2026 timeline.
  - Ship with price leak: Violates hard requirement. Non-viable.
  - Ship without any change notification: Worse UX, potential legal issue (non-refundable policy change without notice).
- **Consequences:** Users know *what* changed but not *how*. Acceptable for Phase 1 if Accom cooperation is delayed. Phase 2 should deliver the full experience with `changeType` field.

---

## 11. Implementation Sequence

### Phase 1a: Foundation (Sprint 1-2)
1. Build Error Aggregation Service in Bundling BE
2. Implement sequential validation orchestration (flight → accom)
3. Implement price containment for **standalone** price changes (both verticals)
4. Build bundling-level opaque price change message
5. Implement action button remapping

### Phase 1b: Combination Error Handling (Sprint 3-4)
6. Negotiate with Accom team for `changeType` + `priceData` fields
7. If Accom delivers: Build Combination Error Sanitizer (strip price, rewrite body/tracker)
8. If Accom not ready: Implement Fallback (suppress "See changes" button + generic banner)
9. Build error display sequence in FE (flight → price → accom non-price)

### Phase 1c: Polish & Testing (Sprint 5)
10. Integration testing across all 8 scenarios
11. Amplitude tracker validation (ensure modified events don't break dashboards)
12. Edge case: what if both flight AND accom have disruptive errors?
13. Edge case: what if accom returns combination error with ONLY price change?

---

## 12. Edge Cases & Risk Register

| # | Edge Case | Handling |
|---|-----------|---------|
| E1 | Both flight sold out AND accom sold out | Flight disruptive error shown (Rule c). Accom not checked. |
| E2 | Flight price up + Accom price down (net zero or negative delta) | Still show price change message. "Your bundled price has been updated." Show the correct direction. |
| E3 | Combination error has price as the ONLY change (no non-price) | After stripping price: combination error becomes empty. Suppress entire combination error. Only show bundled opaque price change. |
| E4 | Accom returns combination error that includes `paymentMethod` change | `paymentMethod` is "Not Relevant" for bundling (PAH excluded). Strip it alongside price. |
| E5 | Accom returns combination error with `maxOcc` (room capacity) change | `maxOcc` is "Not Relevant" for bundling (pax managed by bundling). Strip it. Pass through only genuinely relevant non-price changes. |
| E6 | Flight returns `BOOKING_INTEGRATOR_FAILED` (retry-able) + Accom has price change | Show flight error first (retry option). If user retries and flight succeeds, then show accom-related messages including opaque price. |
| E7 | `changeType` field not present (old Accom API version) | Fallback to degraded mode for that specific error. Log for monitoring. |
| E8 | Price decreased for both flight and accom | Bundling spec says price decrease = silent pass-through. Suppress price message entirely. User sees lower price on payment page. |

---

## Appendix B: Accom Cooperation Request Template

> Draft message to Accommodation PM/Tech Lead

**Subject**: Minimal API addition for Flight+Hotel Bundling error handling

**Ask**: Two small field additions to the `newRatePlanChanges` response in `get-prebook` and `book` APIs:

1. **`changeType`** on each detail item in `popup.details[].details[]`
   - Values: `price`, `cpFlag`, `cpDetail`, `maxOcc`, `travelPolicy`, `mealPax`, `mealFlag`, `valueAdded`, `paymentMethod`
   - You already track these in `tracker.eventDescription` — we just need them as structured data per item
   - This helps us strip price-specific detail from the bottom sheet for bundling orders (opaque pricing requirement)

2. **`priceData`** object on `newRatePlanChanges`
   - `{ oldTotalAAT: number, newTotalAAT: number, currency: string }`
   - You already compute this to generate the price text — we just need the raw values
   - This helps us compute the combined opaque price delta without parsing language-variant text

**Why**: Bundling displays an opaque combined price (flight+hotel). If individual hotel prices leak through the combination error bottom sheet, it breaks our partner agreements. We can't reliably parse price from text because it varies by language.

**Effort estimate**: ~1 sprint based on the fields already being computed internally.

**Timeline**: Need by Sprint X (before bundling Phase 1 launch).

---

## Appendix A: Source References

| Document | Location |
|----------|----------|
| Bundling Initiative Spec | `bundling_initiative_spec.md` |
| Accom Error Code Mapping (Product) | [Confluence HP space - pages/3486548177](https://borobudur.atlassian.net/wiki/spaces/HP/pages/3486548177) |
| Accom Error Code Mapping (TACM - Booking Form) | [Confluence TD space - pages/3814195318](https://borobudur.atlassian.net/wiki/spaces/TD/pages/3814195318) |
| Accom Rate Plan Changes (TACM) | [Confluence TD space - pages/3554510137](https://borobudur.atlassian.net/wiki/spaces/TD/pages/3554510137) |
| Accommodation Error Spec (TSV) | `references/accommodation-error-spec.tsv` |
| Flight Error Spec (TSV) | `references/flight-error-message-error-spec.tsv` |
| Accom Combined Error Screenshot | `references/accom-combined-error-message.png` |
| Accom See Changes Bottom Sheet | `references/accom-see-changes-bottomsheet.png` |
| Accom Sold Out Bottom Sheet | `references/error-message-accom-bottomsheet-sample.png` |
| Flight Sold Out Bottom Sheet | `references/error-message-flight-bottomsheet-sample.png` |
