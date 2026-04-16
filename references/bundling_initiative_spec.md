# Hotel x Flight Bundling Initiative — Comprehensive Specification

> **Source**: [PRD - Hotel x Flight Bundling](https://borobudur.atlassian.net/wiki/spaces/PURE/pages/4416700417/PRD+-+Hotel+x+Flight+Bundling) (Confluence, PURE space)
> **Status**: DRAFT | **Target Release**: Q2 2026 | **Customer**: B2C (Phase 1)
> **Document Owners**: Alwan Mubaidilah, Michelle Elizabeth Amanda Hutasoit, Agung Putra Pratama
> **Last synthesized**: 2026-03-28

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Business Context & Problem Statement](#2-business-context--problem-statement)
3. [Solution Overview](#3-solution-overview)
4. [Phased Scope & Roadmap](#4-phased-scope--roadmap)
5. [End-to-End User Journey](#5-end-to-end-user-journey)
6. [Detailed Requirements by Page](#6-detailed-requirements-by-page)
7. [Price Calculation Logic](#7-price-calculation-logic)
8. [Issuance, Error Handling & Order Lifecycle](#8-issuance-error-handling--order-lifecycle)
9. [Post-Purchase Flows](#9-post-purchase-flows)
10. [Platform Integration](#10-platform-integration)
11. [Dependencies Matrix](#11-dependencies-matrix)
12. [Key Decisions & Clarifications from Cross-Team QnA](#12-key-decisions--clarifications-from-cross-team-qna)
13. [Open Questions & Unresolved Items](#13-open-questions--unresolved-items)
14. [Glossary](#14-glossary)

---

## 1. Executive Summary

**What**: A multi-vertical purchase feature that lets users book **Flight + Accommodation** in a single transaction at a combined "opaque" price, starting with domestic and international round-trip flights paired with hotel stays.

**Why**: Tiket.com currently has no multi-vertical purchase capability. Competitors (Agoda, Expedia, Trip.com, Klook) already offer Flight+Hotel bundles. Internal data shows ~1.1M–1.6M users book multiple travel products but complete only one on Tiket.com due to price, availability, or preference for other platforms. Several accommodation partners are willing to unlock special (opaque) rates when rooms are purchased alongside a flight.

**How**: Introduce a new "Flight x Hotel" vertical owned by the **Order team** that orchestrates existing Flight and Accommodation services. The product creates a single L1 order containing two L2 order details (one Flight, one Hotel) issued sequentially (Flight first, then Hotel). Pricing is displayed as a combined per-pax opaque price with no individual price breakdown exposed to the user.

**Key architectural principle**: This is an **orchestration layer**, not a new monolith. Flight and Accommodation services retain full ownership of inventory, pricing, issuance, and post-purchase logic. The Order/Bundling service handles search orchestration, price combination, booking form unification, and issuance sequencing.

---

## 2. Business Context & Problem Statement

### 2.1 Market Gap

| Dimension | Current State | Desired State |
|-----------|--------------|---------------|
| Multi-vertical purchase | Only multi-room within Accommodation | Flight + Hotel in single checkout |
| Special hotel pricing | Cannot be unlocked | Opaque bundling rates from partners (e.g., Expedia package rates) |
| Competitive parity | Behind Agoda, Expedia, Trip.com, Klook | Feature parity on Flight+Hotel bundles |

### 2.2 User Research Findings

| Segment | Size | Insight |
|---------|------|---------|
| Users booking >1 product but choosing competitors | ~500K–1.1M users | Price and availability drive them elsewhere |
| Median repurchase interval 1–2 days | ~200K users | 37.5% do Flight+Hotel combination |
| Same-day multi-vertical repurchase | ~200K users | Flight+Hotel = ~60% probability |
| Users wanting minimum-spend promo qualification | Qualitative finding | Want single-transaction multi-vertical to qualify |

### 2.3 Core Complications

1. **Unable to unlock hotel special (opaque) prices** — Partners require flight co-purchase as a condition for discounted room rates.
2. **Missed GBV** — Users split purchases across platforms; Tiket.com captures only one leg.
3. **Unmet user need** — Users explicitly want combined booking for promo qualification and convenience.
4. **Flexibility constraints** — Flight and Hotel have fundamentally different discovery flows, guest/passenger models, expiry times, booking forms, payment methods, and add-ons.
5. **Communication complexity** — New booking flow must not overwhelm users; special pricing must be clearly highlighted.

### 2.4 Success Metrics

| Goal | Metric | Notes |
|------|--------|-------|
| Increase Flight + Accom GBV | Baseline/Target TBD | Measured as incremental GBV |
| Adoption rate | X% blended adoption within Y months | `bundling_bookings / total_accom_bookings` and `bundling_bookings / total_flight_bookings` |
| Price competitiveness | TBD | Benchmarked against Agoda/Expedia for same routes |

---

## 3. Solution Overview

### 3.1 Product Name

**Hotel x Flight Bundling** (internal); displayed to users as **"Flight + Hotel"** or **"Flight + Hotel Bundling"**

### 3.2 Core Concept

A **new vertical** at the Order/L1 level that:
- Presents a unified search → SRP → booking → payment flow
- Combines **Flight** (pre-defined by algorithm or user selection) with **Hotel** (user-selected from opaque-price inventory)
- Displays a single **opaque per-pax price** = `(Flight total + Hotel total) / Number of travelers`
- Issues sequentially: **Flight → Hotel** (Hotel issuance only proceeds after Flight succeeds)

### 3.3 Ownership Model

| Concern | Owner |
|---------|-------|
| Bundling Landing Page, SRP, Booking Form | **Order (Bundling) team** — new pages |
| Flight SRP (with tweaks) | **Flight team** — existing page, modified for price-scheme display |
| Flight PRP (with tweaks) | **Flight team** — existing page, modified for fare change |
| Hotel SRP card list | **Order team** — new component within bundling SRP |
| Hotel Room List | **Accommodation team** — existing page, reused |
| Hotel PDP | **Accommodation team** — existing page, reused |
| Payment Page | **Payment team** — existing page |
| My Order / Order Detail | **Order team** — existing, L2 items treated as regular orders |
| Admin Care (CCP) | **Order team + CS tools** — L2 items shown with related-order linking |

### 3.4 Order Data Model

```
L1: Order ID (bundling vertical)
├── L2: Flight Order Detail ID (order_type = FLIGHT)
│   ├── Depart segment
│   └── Return segment (if round trip)
└── L2: Hotel Order Detail ID (order_type = HOTEL)
    └── Room booking
```

- `order_id` (L1) is owned by bundling team as a transaction wrapper
- Each `order_detail_id` (L2) retains its vertical's `item_type`
- Post-payment, L2 items are treated as **independent regular orders** for My Order, refund, reschedule, CS, and post-purchase add-ons
- No new data structure is needed; the existing order hierarchy already supports multiple `order_detail_id` under one `order_id`

---

## 4. Phased Scope & Roadmap

### 4.1 Phase 1 — Core Flight + Hotel Bundling (Target: Q2 2026)

**In Scope:**
- Verticals: Flight, Accommodation
- Customers: B2C only
- Payment: Normal payment only
- Currency: IDR only
- Trip types: Round-trip (default), Single-trip
- New pages: Landing Page, Bundling SRP (Flight pre-defined card + Hotel card list), Booking Form
- Reused pages (with tweaks): Flight SRP, Flight PRP, Hotel Room List, Hotel PDP
- Entry points: Tiket.com homepage icon, Hotel landing page banner, Flight landing page banner, Flight PRP fare card
- Internal dashboard for eligible accommodation and flight pricing rules
- Post-purchase: Refund and reschedule per-vertical (existing flows), PPI, PPA as separate L2 orders
- E-receipt: Single receipt with combined price, no per-item breakdown
- Admin Care: L2-level display with related-order linking

**Out of Scope (Phase 1):**
- Pre-purchase add-ons and insurance in booking form
- 100% refund/reschedule insurance as flight fare (disabled for bundles)
- Down payment and flexible payment
- Pay at hotel
- Different locations for flight destination and hotel (hotel must be at flight destination)
- Different guest/passenger counts between flight and hotel
- Page modules (Upcoming Booking, Continue Payment, Review, etc.) — deferred to post-Phase 1
- B2B Corporate and B2B Affiliate
- Non-IDR currencies
- Multi-city flights
- Tiket Points earning display in SRP/PDP (calculation still happens, just not displayed)

### 4.2 Phase 2 — Add-Ons, Insurance & Payment Expansion

- Pre-purchase add-ons and insurance in booking form
- Down payment enabled
- Hotel location can differ from flight destination (after multi-city flight is enabled)
- Different guest vs. passenger counts
- Free meal/seat selection handling for qualifying fares

### 4.3 Phase 3 — Multi-Vertical Expansion

- Airport Transfer and TTD as additional bundled items
- Broader vertical combinations

---

## 5. End-to-End User Journey

### 5.1 Entry Points → Landing Page / SRP

There are **four** entry points into the bundling flow:

#### A. Tiket.com Homepage → Bundling Landing Page
- New vertical icon (configurable via dashboard, no development needed)
- Global search keyword redirection (needs development)
- User lands on **Bundling Landing Page** with search form

#### B. Hotel Landing Page → Bundling Landing Page
- Banner below hotel search form (configurable assets: text, color, URL via system parameters)
- Click redirects to Bundling Landing Page via deeplink
- Pre-fills: hotel location → `destination_point`, dates → `depart/return`, guests → `travelers`, class defaults to "economy"

#### C. Flight Landing Page → Bundling Landing Page
- Banner below flight search form (same configurability)
- Click redirects to Bundling Landing Page via deeplink
- Pre-fills: origin/destination, dates, passengers → travelers, trip type, class
- If children/infants: popup to confirm ages; If adults >2: popup to confirm room count

#### D. Flight PRP → Bundling SRP (Direct)
- Flight x Hotel fare card appears in PRP fare section alongside regular fares
- Card shows combined per-pax price (flight cheapest fare + estimated hotel)
- Click redirects to **Bundling SRP** directly (skips landing page)
- Pre-fills: chosen flight as pre-defined flight, all search params from flight context
- Experiment variants on card placement (control, var1, var2 with different positions)
- If round trip separate: card only appears in departure stepper
- Infinity promo codes from SRP do NOT apply to the bundle fare card

### 5.2 Bundling Landing Page

**Search Form Fields:**
- Route (departure/destination) — uses Flight service POI API (airport/area/city)
- Date of travel — depart date, return date (round trip), or depart date + nights (single trip)
- Number of rooms and travelers (adults, children 2-17y, infants <2y)
- Class (economy, etc.)
- Nudges below class field (configurable text, colors via Order service)
- Search button with field validation

**Constraints:**
- Default: round-trip setting (configurable via syspar)
- Combination of adults + children max = 7
- Max infants = 4, cannot exceed number of adults
- Room count cannot exceed adult count; auto-calculated as `ceil(adults / 2)` when adults > 2

**Recent Search:**
- Up to 5 past searches displayed
- Click re-executes search; if dates passed: depart = today+2, return = depart+1

**Deeplink Handling:**
- From homepage: auto-fill with latest recent search
- From vertical: handle partial query params per mapping rules above
- Invalid/unavailable params → redirect to Landing Page
- Past dates → auto-adjust per rules above

### 5.3 Bundling SRP (Search Result Page)

The SRP is divided into two main sections loaded **sequentially**: Flight section first, then Hotel section.

#### 5.3.1 Header
- Departure/destination, depart date, return date, rooms, passengers
- Clickable chevron opens **Change Search Modal** (same fields as landing page search form)
- Changes trigger full page reload

#### 5.3.2 Loading Sequence

**When coming from Landing Page:**
1. Order service calls **Flight service** for inventory search
2. Loading animation shown for flight section
3. If flight search **errors**: display dynamic error state from Order service (illustration, title, subtitle, button, URL). No hotel search is attempted.
   - No result → show Change Search Modal
   - Error → reload page
4. If flight search **succeeds**: Order service calculates **pre-defined flight**, shows it in flight card
5. Order service calls **Accommodation service** for inventory search
6. Loading animation shown for hotel section
7. If hotel search **errors**: display similar error state
   - No result → show Change Search Modal
   - Error → reload page
8. If hotel search **succeeds**: display hotel card list

**When coming from Flight PRP:**
1. Pre-defined flight is already set from user's PRP selection (cheapest fare auto-selected)
2. Order service **directly calls Accommodation service** (skips flight search)

#### 5.3.3 Pre-Defined Flight Card

Displays at the top of SRP with a contextual nudge:
- Loading: "Looking for best packages that suits for your trip"
- Loaded (cheapest+direct): "Direct and cheapest"
- Loaded (cheapest+non-direct): "Cheapest"
- User changed flight: "Your preferred flight"
- User changed fare: "Your preferred flight"

**Card contents**: Airline icon, date, class, duration, depart/landing times (+1 if overnight), airports, transit count, baggage info.

**Pre-defined flight selection algorithm:**

| Priority | Rule |
|----------|------|
| 1 | **Direct** flights preferred over non-direct |
| 2 | Among direct: **cheapest** total price |
| 3 | If no direct: **cheapest** regardless of stops |
| 4 | Tie-breaking: **convenient time** |

**Convenient time tie-breaking:**
- **Depart flight**: smallest delta to 12:00 PM departure time; if tied, pick the one **before** noon
- **Return flight**: smallest delta to 12:00 PM departure time; if tied, pick the one **after** noon

**Round-trip selection**: Prioritizes same-airline smart round trip, then cheapest direct combination.

**User actions from pre-defined flight card:**
- **"Change flight"** → redirects to Flight SRP (existing, with tweaks) showing price-scheme deltas
- **"Change benefit"** → redirects to Flight PRP (existing) for fare selection (only appears if >1 fare available for that schedule)
- **"Flight Details" chevron** → bottom sheet with flight detail (existing implementation)

**Exclusions from pre-defined flight:**
- Virtual Interlining (VI) inventory → show empty state
- Fares with 100% refund/reschedule
- Fares with free seat selection
- Fares with free meal selection

#### 5.3.4 Hotel Card List

**Hotel Search Form** (sticky when scrolling past flight section):
- **Date range**: Constrained by flight dates
  - Round trip: check-in ≥ departure date, check-out ≤ return date (Phase 1); user can shorten within window
  - Single trip: check-in ≥ departure date, max 30 nights (Phase 1)
- **Search bar**: Hotel name and area search via Accommodation auto-complete. Area restricted to arrival and transit POI/city.
- **Sort**: Existing Accommodation sort
- **Location filter**: City, area, popular place — restricted to transit/arrival cities
- **Filters**: Price (per-pax level, after tax), Hotel Star, Guest Rating, City, Area, Popular Place
- Phase 1: restricted to refundable hotels/rooms only (configurable via syspar)
- NHA (Non-Hotel Accommodation) is eligible

**Card contents displayed**: Property images, in-policy label, pricing preferred partner badge, USP & benefit, price area, accommodation name, star rating, location, distance to POI, guest rating, review count, review topic extraction.

**Card contents NOT displayed**: Campaign/promo label, Draco score, non-pricing tiket points, login nudges, cross-sell banner, wishlist, timed promo counter, loyalty icon, NHA property type, NHA bedroom spec.

**Sorting logic:**

| Priority | Rule |
|----------|------|
| 1 | Bundling Rate inventory sorted by **biggest rate gap** (biggest discount from normal rate) |
| 2 | Other Rate inventory sorted by **cheapest** |

Phase 1 does not incorporate bundling discount as an ML parameter; subsequent sorting is applied after existing ML results.

**Price display per hotel card:**
```
Per-pax price = (Flight total for all pax + Hotel total for all rooms & nights) / Number of travelers
```

Uses deferred cost pricing. Pay-at-hotel inventory excluded.

**Three card display states**: Hotel with discount, Hotel without discount, Hotel fully booked (note: "fully booked" state confirmed as impossible since sold-out hotels aren't returned by Accommodation service).

#### 5.3.5 Change Date Modal
- Single trip: can extend check-in from departure date onward, max 30 nights
- Round trip: can shorten within flight window only (Phase 1)

### 5.4 Flight SRP (Existing, with Tweaks)

When user clicks "Change flight" from bundling SRP:

**Enabled features**: Existing flight card list, existing filters, existing sorting, campaign countdown, campaign labels.

**Disabled features**: Change search, date picker, reverse origin/destination, search button (desktop), dynamic banner, Infinity promos, floating promo entry point, deferred cost components ("after cashback", slashed price), tiket points earning.

**Price scheme display**: Delta from pre-defined flight, calculated per pax:
```
If available flight is more expensive: + IDR X
If available flight is cheaper: - IDR X
If same price: + IDR 0

Where X = abs(previously_chosen_total - available_total) / number_of_passengers
```

For round trip: departure and return priced independently against their respective pre-defined legs.

Pre-selected flight pinned at top of list. User does NOT choose fare here — Order auto-selects cheapest. After selecting both legs (round trip), user returns to Bundling SRP.

### 5.5 Flight PRP (Existing, with Tweaks)

When user clicks "Change benefit" from bundling SRP:

- Shows fare list with price scheme (upgrade delta per pax)
- Pre-selected fare has selected state
- **Hidden fares**: 100% refund/reschedule, free seat selection, free meal selection
- Fare upgrade per pax calculation example: 3 adults + 1 child + 1 infant, upgrade = IDR 300K/adult → displayed as `+ IDR {{300,000 × 3 / 5}}`
- Round trip separate: depart & return steppers shown; user selects both before returning to bundling SRP

### 5.6 Hotel Room List & PDP (Existing, Reused)

**Disabled features**: Change date, rooms, guests, change button, tiket points earning display.

**Sorting**: Bundling rate (biggest gap) → other rate (cheapest). Pay-at-hotel excluded.

**Constraint**: User cannot choose more than one room type (enforced in FE Bundling).

**Validation**: Existing Accommodation validations (room selection → booking form) are NOT applied in bundling flow. All validation deferred to Booking Form → Payment Page transition.

### 5.7 Booking Form

**Summary section**: Single card with Flight and Hotel subsections + Baggage & Allowance Policies link.

**Contact details form**: Salutation, name, phone, email. Smart Profile integration. Member data autofill.

**Passenger/guest details**: One form per flight passenger. Hotel guest = subset of flight passengers. Each room needs one adult representative for check-in. Each adult can only represent one room.

**Hotel special requests**: Checkbox + free text, forwarded to Accommodation.

**Price detail footer**: Combined bundled price only. Fees and taxes combined (not separated by vertical). No per-item breakdown.

**Back button behavior**: User chooses between returning to Bundling SRP (preserving selections) or staying in form.

### 5.8 Payment Page

- Order summary for both items
- Payment time limit: shorter of flight or hotel limits
- Phase 1: Normal payment methods only (all currently available methods)
- Pay at Hotel disabled
- No per-item price breakdown; bundled price only
- Tech solutioning must be scalable for down payment and flexible payment (Phase 2)

---

## 6. Detailed Requirements by Page

*See Section 5 for the full page-by-page breakdown. This section captures additional nuances.*

### 6.1 Deeplink Contract

The bundling flow relies heavily on deeplinks for cross-vertical entry points. Query parameters must include:

| Parameter | Source: Hotel LP | Source: Flight LP | Source: Flight PRP |
|-----------|-----------------|-------------------|-------------------|
| Trip type | round_trip (default) | from toggle | from flight type |
| Departure point | empty (user must fill) | origin | origin |
| Destination point | hotel location | destination | destination |
| Depart date | start_date | departure_date | departure_date |
| Return date | end_date | return_date | return_date |
| Nights | calculated | calculated / 1 (one-way) | calculated / 1 |
| Travelers | guests | passengers | passengers |
| Child ages | if present | popup if present | popup if present |
| Rooms | from hotel form | auto-calc or popup if adults>2 | auto-calc or popup |
| Class | economy (default) | seat_class | seat_class |
| Pre-defined flight | N/A | N/A | selected flight(s) + cheapest fare |

### 6.2 Error State Contract

All error states are dynamic from Order service backend. Client renders based on a standard contract:

```typescript
interface ErrorState {
  illustration: string;   // URL or asset key
  title: string;
  subtitle: string;
  buttonText: string;
  redirectUrl: string;
}
```

Error categories in booking flow:
1. **Inventory unavailable** (disruptive) — redirects to SRP
2. **Price increase** (informational) — popup, user can proceed
3. **Price decrease** — no popup, silently applied
4. **Benefit change** — TBD
5. **Booking timeout** — expiry handling
6. **General error** — retry/redirect

Multiple errors are **stacked** in order of occurrence.

---

## 7. Price Calculation Logic

### 7.1 Core Formula

```
Per-pax bundled price = (Flight total price for ALL passengers + Hotel total price for ALL rooms & ALL nights) / Number of travelers (including children and infants)
```

### 7.2 Price in Flight PRP Fare Card (Entry Point)

```
Bundled price per pax = (Flight cheapest fare total + Hotel first-in-list total) / Number of travelers
Booked-separately price = Bundled price × 125 / 100
```

The "booked-separately" price is a synthetic comparison price (25% markup).

### 7.3 Price Scheme in Flight SRP (Change Flight)

```
Delta = (Available flight total - Previously chosen flight total) / Number of passengers
Display: "+ IDR X" or "- IDR X" or "+ IDR 0"
```

Round trip: departure and return calculated independently.

### 7.4 Price Scheme in Flight PRP (Change Benefit/Fare)

```
Fare upgrade delta per pax = (Upgrade amount per adult × Number of adults) / Total passengers
```

Fare upgrades only apply to adults but the per-pax display divides by ALL passengers.

### 7.5 Deferred Cost & Kuber

- Bundling flow uses deferred cost pricing (cashback shown in price)
- Each vertical calls Kuber independently with additional `is_bundling` flag
- Phase 1: Existing vertical budget pockets (confirmed with corpstrat/commercial — no separate bundle budget)
- Phase 2 consideration: Orchestrator-level Kuber call with bundle vertical ID spanning multiple verticals

### 7.6 Tax and Fees

- Displayed as a **combined** figure for both flight and hotel
- Not separated per vertical in any user-facing context
- Convenience fee: stored per vertical in their respective cart/collection tables; not in Order DB

---

## 8. Issuance, Error Handling & Order Lifecycle

### 8.1 Issuance Sequence

```
User pays → Order service triggers Flight issuance
  ├── Flight issuance FAILS → Auto-refund BOTH flight AND hotel → Both transition to BU status
  └── Flight issuance SUCCEEDS → Order service triggers Hotel issuance
        ├── Hotel issuance FAILS → Partial refund for hotel ONLY (auto-refund after 3 retries for B2C)
        │   Flight remains issued
        └── Hotel issuance SUCCEEDS → Order status updated, user sees both vouchers
```

### 8.2 Key Issuance Decisions

- **No hold-and-resume**: The original plan to use Expedia's hold/resume API was **struck through** in the PRD. Hotel issuance proceeds directly after flight success. (Comment from Accommodation PM confirmed this should not be limited to Expedia inventory.)
- **Sequential, not parallel**: Flight MUST succeed before hotel issuance begins.
- **Partial issuance is possible**: Flight success + hotel failure = partially issued order.

### 8.3 Validation Flow

All validations are collated into a **single validation pass** at the Booking Form → Payment Page transition:

1. **Inventory check**: Order service validates with both Flight and Accommodation services
   - Hotel unavailable → redirect to SRP (flight preserved as pre-defined)
   - Flight unavailable → redirect to SRP (hotel on top of list)
2. **Price revalidation**: Comparison of current vs. original prices
   - Total increases → informational popup
   - Total decreases → silent pass-through
3. **Room representative validation**: Each room must have one adult; each adult can only represent one room

### 8.4 Order Status Lifecycle

```
CREATED → WAITING_PAYMENT → PAID → ISSUING
  ├── ISSUED (both success)
  ├── PARTIALLY_ISSUED (flight success, hotel fail) → hotel auto-refund
  └── FAILED (flight fail) → both auto-refund → BU status
```

Post-payment, each L2 order detail transitions independently following its vertical's existing status machine.

---

## 9. Post-Purchase Flows

### 9.1 Refund

- Follows **per-vertical existing policy** at L2 (order detail) level
- User can refund hotel while keeping flight, or vice versa
- Refund amount for hotel uses the opaque amount (not the normal rate)
- No "bundle-level" single refund — each item refunded independently
- Confirmed in QnA: "refund will be done per order detail id level where accommodation order detail and flight order detail will preserve exactly like current Smart trip refund"

### 9.2 Reschedule

- Follows per-vertical existing policy at L2 level
- Independent per item

### 9.3 Post-Purchase Insurance (PPI)

- Enabled per L2 item after order is fully paid
- Creates separate L2 order, linked as related order
- Existing implementation; no bundle-specific changes

### 9.4 Post-Purchase Add-Ons (PPA)

- Same as PPI: enabled per L2 item, separate L2 order, existing implementation
- **Note from Accommodation PM**: Pre-purchase add-ons are NOT available in Phase 1 booking form

### 9.5 Loyalty / Tiket Points

- Earn rate: `(BASE_PRICE + PROMO_CODE + GIFT_VOUCHER + TIX_POINT) × 1%`
- Combined from both verticals
- If one item fails issuance, points earned only for successfully issued item
- Instant Point Cashback (IPC) revocation on partial refund: revoked at **order_id level** (all points revoked, not proportional to refunded item)
- Tiket points earning display is suppressed in SRP/PDP during Phase 1

---

## 10. Platform Integration

### 10.1 My Order

- L1 bundling order: single card when **unpaid**
- L2 items: separate cards when **paid**, following existing vertical display
- Pre-payment cancellation follows existing flow
- Order title and icon mapped internally (TBC)

### 10.2 E-Receipt

- **Single receipt** covering both flight and hotel
- Product name: "Flight + Hotel Bundling"
- Contains: Order ID, order name, email, phone, paid date, payment method, description (per-item details allowed as long as no per-item price breakdown), subtotal, admin fee, total payment
- Same receipt displayed in both Flight and Hotel L2 order details

### 10.3 E-Voucher

- Accessed per L2 order detail (flight voucher in flight order, hotel voucher in hotel order)
- Follows existing implementation per vertical

### 10.4 Notifications (WA & Email)

- Success/failure notifications — exact content TBC
- Three scenarios: both issued, only flight issued (hotel failed), both failed
- No per-item price breakdown in notifications

### 10.5 Promo

- Bundling flag added to `dynamicPromo` object
- Promo system consumes both `orderType` and `productType` (from Payment) — needs logic adjustment
- Promo creation: must select both flight AND hotel verticals; new criterion in dynamic promo
- All current promo functionality (including promo detail page) preserved for bundling

### 10.6 Payment

- Vertical recognition: via `paymentUnitDetail.productType[0]` — first element of payment unit detail product type array
- Not creating a brand-new vertical at payment level; preserving existing vertical identifiers at order detail level
- Payment team needs to adjust how they identify bundling orders

### 10.7 Financial Dashboard / CFD

- Each bundling transaction recorded per vertical following existing flow
- Additional flag differentiates bundle vs. standalone
- No separate price components owned by bundling team — all sourced from verticals and/or payment

### 10.8 Admin Care (CS Dashboard)

- Search by order ID shows L2 items (flight card + hotel card)
- Each item displays same info and actions as standalone orders
- **Related orders** section links flight and hotel items within the same bundle
- Invoice and paid amount covers full bundling amount
- Payment history from Payment team
- Receipt and e-voucher modals same as consumer-facing

### 10.9 Data / BI

- Bundling data accessible via BigQuery with same flows + bundling flag
- Order counting: `order_id` represents one checkout; vertical-level `count(distinct order_id)` still works per vertical, but cross-vertical aggregation shows 1 order
- Promo, GV, subsidy split: discussion with commercial/corpstrat scheduled

### 10.10 Insurance

- Phase 1: 100% refund/reschedule insurance **disabled** as flight fare option in bundles
- Phase 1: Insurance excluded from booking form
- Phase 2: Insurance as add-on in booking form
- Flight team needs parameter to differentiate bundle vs. standalone for insurance logic

### 10.11 Discovery / Homepage

**Phase 1 scope (confirmed with Discovery PM):**

| Module | Status | Notes |
|--------|--------|-------|
| Vertical Icon | No dev, dashboard config only | Request via form, align with other verticals |
| Global Search (keyword redirect) | Needs development | Keywords TBD |
| Global Search (query in search) | Needs development | |
| Upcoming Booking | Test during integration | Expectation: seamless, no changes on flight/hotel post-purchase data |
| Continue Payment | Need figma design preview | Uses existing `/tix-order-composer/yourorder/v1/waitingpaymentlist` — no changes planned |
| Review Widget | No UI changes, needs data verification | Hotel reviews from bundling bookings |
| Promo List | No dev, need testing | Order PM creates new Flight+Hotel promo category |
| Page modules (Inventory, Contextual) | **Out of scope Phase 1** | Deferred per discussion with Discovery PM |
| Cross-Sell GV | **Out of scope Phase 1** | |
| Recent Search | **Out of scope Phase 1** | |

---

## 11. Dependencies Matrix

| Team | Dependent? | Key Deliverables | Development Required? |
|------|-----------|------------------|----------------------|
| **Accommodation** | **Yes** | Entry point banner, bundling rate inventory, search/filter/sort, hold→issuance flow, fallback handling, promo objects, post-purchase flag | Yes |
| **Flight** | **Yes** | Entry point (LP banner + PRP fare card), inventory with price scheme, PRP fare tweaks, differentiate bundle vs standalone, promo objects, post-purchase flag | Yes |
| **Payment** | **Yes** | Bundling order recognition via productType, receipt generation, payment history API | Yes |
| **Promo** | **Yes** | `dynamicPromo` flag support, new criterion for bundling, orderType+productType logic adjustment | Yes |
| **Insurance** | **Yes (Phase 1 partial)** | Disable 100% R/RR fare for bundles; Phase 2: booking form insurance | Yes (Flight to pass bundle flag) |
| **Loyalty** | **Yes** | Point earning calculation (combined verticals), IPC revocation handling | Yes (depends on corpstrat answer) |
| **Discovery** | **No** | Vertical icon config, global search, page modules (Phase 2+) | Partial (global search query) |
| **Refund** | **No** | Existing per-vertical flow works | No (depends on solution) |
| **Member** | **No** | Login banner, smart profile, profile creation, autofill, UNM validation | No |
| **Pricing** | **No** | Kuber evaluation with bundling flag, deferred cost strategy | No |
| **CFD** | **No** | Same flow + bundling flag | No (depends on solution) |
| **Data** | **No** | BQ access with bundling flag | No (depends on solution) |
| **B2B Corporate** | **No** | Validation and testing only (Phase 1 excluded) | No |
| **B2B Affiliate** | **No** | No plan for API channel bundling; possibly offline tour agent in 2027 | No |
| **TTD** | **No** | Review and feedback only (Phase 3 inclusion) | No |
| **NFT** | **No** | Review and feedback only (Phase 3 inclusion) | No |
| **Growth** | **No** | Cross-sell recommendation (future) | No |
| **CS & Help Center** | **No** | Knowledge sharing, help center content | No |

---

## 12. Key Decisions & Clarifications from Cross-Team QnA

### 12.1 Order Structure

> **Q**: Will bundling introduce a completely new order data structure?
> **A**: No. The existing structure already supports multiple `order_detail_id` (L2) under one `order_id` (L1). Examples: train+insurance, flight+insurance. No backward-incompatible changes.

### 12.2 Refund Granularity

> **Q**: Will refund be per-bundle or per-item?
> **A**: Per order_detail_id level. Flight and hotel refunds operate independently, exactly like current smart-trip refund behavior.

### 12.3 Pay at Hotel

> **Q**: Is Pay at Hotel applicable for packaged rates?
> **A**: Even if a vendor returns PAH for a package rate, it will be filtered out. PAH is out of scope for bundling.

### 12.4 Hotel Search Area Restriction

> **Q**: Can accommodation search results include nearby areas?
> **A**: Accommodation uses "Lepus" (lat/long within 10km radius) for POI/near-me searches and "Fornex" (atlas ID) for city-level searches. POI is generalized to city level for bundling.

### 12.5 Guest Names for Hotel

> **Q**: Should every flight passenger be sent to accommodation provider?
> **A**: One PIC (person in charge) per room is sufficient. PIC should be selected from the passenger list.

### 12.6 B2B Affiliate

> **Q**: Will B2B Affiliate adopt bundling?
> **A**: No plan for API channel bundling. Possibly offline tour agent in 2027. B2B partners have stricter SLAs; solution should be separate from B2C.

### 12.7 Subsidy Budget

> **Q**: Will bundle subsidies use existing vertical budgets or a new bundle budget?
> **A**: Existing vertical budget pockets (agreed with corpstrat and commercial). No dedicated bundle budget for Phase 1.

### 12.8 Convenience Fee

> **Q**: How will convenience fee be handled for multi-vertical orders?
> **A**: Vertical convenience fee is stored in each vertical's cart/collection table. For additional fees from Payment team, stored as consolidated field. Allocation depends on whether evaluation is at order_id or order_detail_id level.

### 12.9 Insurance Linkage

> **Q**: Insurance is currently linked at order_id level. How to identify which vertical it belongs to?
> **A**: Follow-up needed with Insurance PM on whether each insurance product (credential) is bound to a vertical entity.

### 12.10 Promo/GV/Subsidy Split

> **Q**: Promo, GV, and subsidy are at order_id level. How to split across verticals?
> **A**: Discussion scheduled with commercial, corpstrat, and data team. Solution TBD.

### 12.11 Loyalty IPC Revocation

> **Q**: How does IPC work on partial refund?
> **A**: Points revoked at order_id level (all points revoked regardless of which item is refunded). If points already used, deducted from refund amount.

### 12.12 B2B Corporate Travel Policy

> **Q**: How are travel policy checks handled?
> **A**: Flight and Hotel services return inventory based on B2B corp user's account properties. No additional bundling-specific development needed for travel policy.

### 12.13 Sequential Issuance & Hotel Failure

> **Q**: What if flight is issued but hotel issuance fails?
> **A**: Hotel implements auto-refund for B2C after 3 retry failures. Same failure information shown in accommodation card. The opaque hotel amount is refunded.

### 12.14 SEO & Slugs

> **Q**: Will bundling need new SEO templates and sitemaps?
> **A**: Discussion session created; slug format TBD (e.g., `/flight-hotel/`, `/bundling/`, `/package/`).

---

## 13. Open Questions & Unresolved Items

### 13.1 From PRD (marked TBC/TBD)

| Item | Context | Status |
|------|---------|--------|
| Slashed price formula in SRP | What is the "booked separately" comparison price? | TBC — 125% of bundle price is mentioned but not finalized |
| Price detail in fare bottom sheet | What price components shown when user clicks "see detail" in PRP fare card | TBD |
| Children handling from Flight LP deeplink | If children/infants come from flight LP, should bundling LP force age input before proceeding? | Open inline comment |
| Error handling for room selection in Room List | What happens when all rooms are sold out and user is redirected? Should flight be preserved? | Open inline comment |
| Flight chosen preservation on change | When user changes flight in SRP, should the previous flight selection be preserved for back-navigation? | Open inline comment |
| Notification content | Exact WA/email notification copy for success, partial failure, full failure | TBC |
| Order title and icon mapping | How bundling orders appear in My Order list | TBC |

### 13.2 From Inline Comments (Unresolved)

| Comment | Author Context | About |
|---------|---------------|-------|
| "apakah chosen flight(s) bisa dipreserved dan dijadikan sebagai pre-defined flight" | FE engineer | Flight preservation when coming from landing page with pre-selected flight context |
| "confirm to pm accom the logic" on room count calculation | PM | Whether accommodation's existing room-count logic matches bundling's `ceil(adults/2)` rule |
| "nggak ngerti examplenya" on first flight changes pricing | Flight PM | Clarity needed on the price scheme example in the PRD |
| "ini ga consider non-direct properties lain ya?" on cheapest non-direct selection | Flight PM | Should pre-defined flight consider transit count and transit time, not just price? |
| "no, in SRP flight currently we only show price per pax based on the adult price" | Flight PM | Potential clash between flight's adult-only per-pax display and bundling's all-pax display |
| "i believe this should not be the case" re: Expedia hold API | Accom PM | Confirmed: should NOT be limited to Expedia; hold/resume approach removed |
| "i believe we still cannot purchase post purchase adds on" | Accom PM | Post-purchase add-ons typically happen before payment; clarification needed on post-payment add-on flow |
| Price calculation for hotel tax/fee | FE engineer | Whose responsibility is tax aggregation: Accommodation or Order service? |
| Issuance confirmation flow | FE engineer | Is it Flight→Hotel direct or Flight→Order→Hotel? |
| "related orders" in refund page | Refund PM | Should related bundle items show when user requests refund on one item? |

### 13.3 From Footer Comment (Scope Decisions Pending)

- **Upcoming Booking** module: show booking from flight+hotel → needs testing
- **Continue Payment** module: show booking from flight+hotel → needs figma design preview
- **Review**: show reviews from hotel in flight+hotel bookings → needs testing
- **Promo list**: show promos from flight+hotel → needs testing
- **GV**: show GV from flight+hotel → out of scope Phase 1

---

## 14. Glossary

| Term | Definition |
|------|-----------|
| **L1 / Order ID** | Top-level order entity representing a single checkout transaction. Can contain multiple L2 items. |
| **L2 / Order Detail ID** | Individual vertical-specific order item within an L1 order. Each has its own `item_type` (FLIGHT, HOTEL, etc.). |
| **Opaque Price** | Combined bundled price that does not reveal the individual flight or hotel cost to the user. |
| **Bundling Rate** | Special accommodation rate offered by partners (e.g., Expedia package rates) that is only available when booked together with a flight. |
| **Pre-defined Flight** | Algorithm-selected flight displayed automatically in the bundling SRP before user interaction. |
| **Price Scheme** | Delta-based pricing display in Flight SRP showing `+/-` relative to the pre-defined flight. |
| **PRP** | Product Review Page (Flight) — where users see fare details and can select different fare classes. |
| **SRP** | Search Result Page — inventory listing page. |
| **PDP** | Product Detail Page (Hotel) — individual hotel detail view with room list. |
| **Kuber** | Tiket.com's pricing/promo evaluation engine that handles subsidy, cashback, and promotion logic. |
| **Deferred Cost** | Pricing strategy where cashback is already reflected in the displayed price. |
| **Smart Round Trip** | Flight pricing model where departure and return are priced as a package from the same airline. |
| **VI (Virtual Interlining)** | Combining flights from different airlines into a single itinerary — excluded from bundling Phase 1. |
| **SSRR** | Self-Service Rebooking/Refund — CS tool for rebooking failed hotel orders. |
| **BU** | Business Unit status — transitional status for failed issuance before auto-refund. |
| **PAH** | Pay at Hotel — payment method where guest pays at property; excluded from bundling. |
| **IPC** | Instant Point Cashback — tiket points earned immediately and potentially revoked on refund. |
| **PPI** | Post-Purchase Insurance — insurance bought after order payment as separate L2 order. |
| **PPA** | Post-Purchase Add-ons — add-ons bought after order payment as separate L2 order. |
| **NHA** | Non-Hotel Accommodation — property types like villas, apartments, hostels. |
| **CFD** | Centralized Finance Dashboard — financial ledger and reconciliation system. |
| **Syspar** | System parameter — server-side configuration that can be changed without deployment. |
| **Fornex** | Accommodation's ML-based search engine using atlas IDs for city-level and above searches. |
| **Lepus** | Accommodation's proximity-based search engine using lat/long within 10km radius. |

---

> **For AI agents reading this document**: This spec represents the Hotel x Flight Bundling initiative for Tiket.com's Order Platform. The Order/Bundling team acts as an orchestration layer over existing Flight and Accommodation microservices. Key invariants: (1) opaque combined pricing with no per-item breakdown to users, (2) sequential Flight→Hotel issuance, (3) L2 items treated as independent regular orders post-payment, (4) existing vertical services retain full domain ownership. When analyzing this initiative's impact on the Order Platform architecture (status tracking, price breakdown, financial reconciliation), note that bundling introduces a new L1 order type but does NOT change the fundamental L2 data model or vertical service boundaries.
