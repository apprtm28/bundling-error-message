# Hotel x Flight Bundling Initiative — Comprehensive Specification

> **Source**: [PRD - Hotel x Flight Bundling](https://borobudur.atlassian.net/wiki/spaces/PURE/pages/4416700417/PRD+-+Hotel+x+Flight+Bundling) (Confluence, PURE space)
> **Child pages**: [Cross Team QnA](https://borobudur.atlassian.net/wiki/spaces/PURE/pages/4565336086) · [Price Aggregation Documentation](https://borobudur.atlassian.net/wiki/spaces/PURE/pages/4744970410) · Error Message in Booking Process Documentation (draft)
> **Status**: DRAFT | **Target Release**: Q2 2026 | **Customer**: B2C, B2B Corporate (Phase 1)
> **Document Owners**: Alwan Mubaidilah, Michelle Elizabeth Amanda Hutasoit, Agung Putra Pratama
> **Confluence version**: 99 (as of 2026-04-15)
> **Last synthesized**: 2026-04-16

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
15. [Reference Documents](#15-reference-documents)

---

## 1. Executive Summary

**What**: A multi-vertical purchase feature that lets users book **Flight + Accommodation** in a single transaction at a combined "opaque" price, starting with domestic and international round-trip flights paired with hotel stays.

**Why**: Tiket.com currently has no multi-vertical purchase capability. Competitors (Agoda, Expedia, Trip.com, Klook) already offer Flight+Hotel bundles. Internal data shows **~1.1M–1.6M users** book multiple travel products but complete only one on Tiket.com due to price, availability, or preference for other platforms. Accommodation partners are willing to unlock special (opaque) rates when rooms are purchased alongside a flight.

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
| Users booking >1 product but choosing competitors | ~500K users | Price and availability drive them elsewhere |
| Users who book >1 product on competitors (favorite platform) | ~570K–1.1M users | Platform preference; total missed opportunity ~1.1M–1.6M |
| Median repurchase interval 1–2 days | ~200K–320K users | 37.5% do Flight+Hotel combination |
| Same-day multi-vertical repurchase | ~200K users | Flight+Hotel = ~60% probability; Flight/Train+Hotel ~73%; +AT ~77%; +TTD ~82% |
| Users wanting minimum-spend promo qualification | Qualitative finding | Want single-transaction multi-vertical to qualify |

### 2.3 Core Complications

1. **Unable to unlock hotel special (opaque) prices** — Partners require flight co-purchase as a condition for discounted room rates.
2. **Missed GBV** — Users split purchases across platforms; Tiket.com captures only one leg.
3. **Unmet user need** — Users explicitly want combined booking for promo qualification and convenience.
4. **Flexibility constraints** — Flight and Hotel have fundamentally different:
   - Discovery flows and search parameters
   - Guest/passenger models (pax vs. guests)
   - Expiry times
   - Booking forms
   - Payment methods (flight DP vs. hotel Pay at Hotel / flexible)
   - Vertical-specific add-ons
   - Post-purchase partial refund/reschedule scenarios
5. **Communication complexity** — New booking flow must not overwhelm users; special pricing must be clearly highlighted with low cognitive load.

### 2.4 Success Metrics

| Goal | Metric | Notes |
|------|--------|-------|
| Increase Flight + Accom GBV | Baseline/Target TBD | Measured as incremental GBV |
| Adoption rate | Baseline 0% → Target X% blended within Y months | `bundling_bookings / total_accom_bookings` and `bundling_bookings / total_flight_bookings` |
| Price competitiveness (Accommodation) | TBD | Benchmarked against Agoda/Expedia for same routes |
| Price competitiveness (Flight) | TBD | Benchmarked against Agoda/Expedia for same routes |

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

- `order_id` (L1) is owned by bundling team as a transaction/checkout wrapper
- Each `order_detail_id` (L2) retains its vertical's `item_type`
- Post-payment, L2 items are treated as **independent regular orders** for My Order, refund, reschedule, CS, and post-purchase add-ons
- No new data structure is needed; the existing order hierarchy already supports multiple `order_detail_id` under one `order_id` (existing examples: train+insurance, flight+insurance)
- No separate price component is owned by bundling team — all sourced from verticals and/or payment

### 3.5 Constraints & Key Decisions (Phase 1)

| Topic | Phase 1 Decision | Phase 2 |
|-------|-----------------|---------|
| **Discovery** | New SRP (Order-owned); flight pre-defined, hotel not pre-defined; hotels at flight destination only; entry points: Flight LP + PRP, Hotel LP, Homepage | Hotel location can differ from flight destination (after multi-city flight exists) |
| **Pax vs Guests** | Guest count follows passengers; same passenger data for both; user can pick # rooms (rooms ≤ adults) | Different guest vs. passenger counts; mismatched names allowed |
| **Expiry** | Shorter of flight vs. hotel expiry; surface changes on booking/payment per current behavior | — |
| **Booking Form / Issuance** | Merged Order-owned booking form; ~~Hotel will hold booking using Expedia hold API~~; sequence: Flight → Hotel; partial refund for hotel only if hotel issuance fails after successful flight | — |
| **Payment** | Normal payment only | Down Payment enabled with repayment and auto-cancel flows |
| **Add-ons** | Post-purchase add-ons per vertical at L2 after full pay | Pre-purchase add-ons + insurance in booking form |
| **Post-purchase** | Refund/reschedule per vertical; one receipt row (Flight+Hotel) without price breakdown; PPI/PPA per vertical | — |
| **Child Age** | **2–11 years** (changed from previous 2–17) | — |

---

## 4. Phased Scope & Roadmap

### 4.1 Parameter Table

| Parameter | In Scope | Out of Scope |
|-----------|----------|-------------|
| **Verticals** | Flight, Accommodation, NFT (AT), TTD, Insurance | Other verticals |
| **Customers** | B2C, B2B Corporate | B2B Affiliate |
| **Payment** | Normal; Down payment (per config & testing) | Flexible payment |
| **Currency** | IDR | Others (architecture should allow future extension) |

### 4.2 Phase 1 — Core Flight + Hotel Bundling (Target: Q2 2026)

**In Scope:**
- Verticals: Flight, Accommodation
- Customers: B2C, B2B Corporate
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
- Coach marks for first-time users (Change benefit, Change date)
- SEO: New templates and URL slugs for bundling pages
- Deeplinks from all entry points with parameter mapping

**Out of Scope (Phase 1):**
- Pre-purchase add-ons and insurance in booking form
- 100% refund/reschedule insurance as flight fare (disabled for bundles)
- Flexible payment
- Pay at hotel
- Different locations for flight destination and hotel (hotel must be at flight destination)
- Different guest/passenger counts between flight and hotel
- Page modules: Inventory, Contextual — deferred to post-Phase 1
- Cross-sell GV — deferred
- Recent Search on homepage — deferred
- B2B Affiliate
- Non-IDR currencies
- Multi-city flights
- Tiket Points earning display in SRP/PDP (calculation still happens, just not displayed)
- Free meal/seat selection handling (fares with these are excluded)

### 4.3 Phase 2 — Add-Ons, Insurance & Payment Expansion

- Pre-purchase add-ons and insurance in booking form
- Down payment enabled with **repayment** and **auto-cancel** flows
- Hotel location can differ from flight destination (after multi-city flight is enabled)
- Different guest vs. passenger counts and mismatched names
- Free meal/seat selection handling for qualifying fares
- Hotel stay window can extend beyond flight dates (up to 365 nights, configurable)

### 4.4 Phase 3 — Multi-Vertical Expansion

- Flight + Accommodation + **Airport Transfer (AT)** + **TTD** as additional bundled items
- Broader vertical combinations

---

## 5. End-to-End User Journey

### 5.1 Entry Points → Landing Page / SRP

There are **four** entry points into the bundling flow:

#### A. Tiket.com Homepage → Bundling Landing Page
- New vertical icon (configurable via dashboard, no development needed — request via Google Form to Discovery, align with #homapagecurator Slack)
- Global search keyword redirection (needs development)
- Global search query support (needs development)
- User lands on **Bundling Landing Page** with search form

#### B. Hotel Landing Page → Bundling Landing Page
- Banner below hotel search form (configurable assets: text, color, URL via system parameters; changed from checkbox to banner)
- Click redirects to Bundling Landing Page via deeplink
- Pre-fills: hotel location → `destination_point`, dates → `depart/return`, guests → `travelers`, class defaults to "economy"

#### C. Flight Landing Page → Bundling Landing Page
- Banner below flight search form (same configurability; changed from checkbox to banner)
- Click redirects to Bundling Landing Page via deeplink
- Pre-fills: origin/destination, dates, passengers → travelers, trip type, class
- If children/infants: popup to confirm ages (defaults: child age 11 years, infant 1 year)
- If adults >2: popup to confirm room count

#### D. Flight PRP → Bundling SRP (Direct)
- Flight x Hotel fare card appears in PRP fare section alongside regular fares (moved from Flight SRP to Flight PRP)
- Card shows combined per-pax price (flight cheapest fare + estimated hotel)
- Click redirects to **Bundling SRP** directly (skips landing page)
- Pre-fills: chosen flight as pre-defined flight, all search params from flight context
- Experiment variants on card placement (control, var1, var2 with different positions)
- If round trip separate: card only appears in departure stepper; price includes departure + return
- Infinity promo codes from SRP do NOT apply to the bundle fare card

### 5.2 Bundling Landing Page

**Search Form Fields:**
- Route (departure/destination) — uses Flight service POI API (airport/area/city); destination localized to Atlas ID for hotel search
- Date of travel — depart date, return date (round trip), or depart date + nights (single trip)
- Number of rooms and travelers (adults, **children 2–11y**, infants <2y)
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
- From homepage: auto-fill with latest recent search; default round trip if no params
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
- Schedules whose **only** fares are excluded types → schedule excluded entirely

#### 5.3.4 Hotel Card List

**Hotel Search Form** (sticky when scrolling past flight section):
- **Date range**: Constrained by flight dates
  - Round trip: check-in ≥ departure date, check-out ≤ return date (Phase 1); user can shorten within window; default nights = window between departure arrival and return departure
  - Single trip: check-in ≥ departure date, default 1 night from departure arrival, max 30 nights (Phase 1); future: up to 365 nights (configurable)
- **Search bar**: Hotel name and area search via Accommodation auto-complete. Area restricted to arrival and transit POI/city.
- **Sort**: Existing Accommodation sort
- **Location filter**: City, area, popular place — restricted to transit/arrival cities
- **Filters**: Price (per-pax level, after tax), Hotel Star, Guest Rating, City, Area, Popular Place
- Phase 1: restricted to refundable hotels/rooms only (configurable via syspar)
- NHA (Non-Hotel Accommodation) is eligible

**Card contents displayed**: Property images, in-policy label (B2B corp), pricing preferred partner badge, USP & benefit, price area, accommodation name, star rating, location, distance to POI, guest rating, review count, review topic extraction.

**Card contents NOT displayed**: Campaign/promo label, Draco score, non-pricing tiket points, login nudges, cross-sell banner, wishlist, timed promo counter, loyalty icon, NHA property type, NHA bedroom spec.

**Sorting logic:**

| Priority | Rule |
|----------|------|
| 1 | Bundling Rate inventory sorted by **biggest rate gap** (biggest discount from normal rate) |
| 2 | Other Rate inventory sorted by **cheapest** |

Phase 1 does not incorporate bundling discount as an ML parameter; sorting is applied **after** existing ML results. No new ML model for P1 — only price engine. Default + manual filter. Parallel pipeline: get recommendation + get price. Future phase: new ML with bundling discount + airline as parameters.

**Price display per hotel card:**
```
Per-pax price = (Flight total for all pax + Hotel total for all rooms & nights) / Number of travelers
```

Uses deferred cost pricing. Pay-at-hotel inventory excluded. Round up when division is uneven.

**Three card display states**: Hotel with discount, Hotel without discount, Hotel fully booked (note: "fully booked" state confirmed as impossible since sold-out hotels aren't returned by Accommodation service).

**Coach Marks** (first-time users): Highlights for **Change benefit** and **Change date** actions.

#### 5.3.5 Change Date Modal
- Single trip: can extend check-in from departure date onward, max 30 nights (Phase 1); future: up to 365 nights (configurable)
- Round trip: can shorten within flight window only (Phase 1); future: can extend beyond range up to 365 nights (configurable)
- Only available when **both** flight and hotel inventory have loaded

#### 5.3.6 Hotel Search Details Modal
- Triggered when adults > 2 or children/infants come from Flight LP/PRP deeplinks
- Allows confirmation of rooms and child/infant ages
- Defaults: child age **11 years**, infant age **1 year**
- Applies to deeplinks with predefined passengers

### 5.4 Flight SRP (Existing, with Tweaks)

When user clicks "Change flight" from bundling SRP:

**Enabled features**: Existing flight card list, existing filters, existing sorting, campaign countdown, campaign labels.

**Disabled features**: Change search, date picker, reverse origin/destination, search button (desktop), dynamic banner, Infinity promos, floating promo entry point, deferred cost components ("after cashback", slashed price), tiket points earning.

**Price scheme display**: Delta from pre-defined flight, calculated per pax:
```
If available flight is more expensive: + IDR X
If available flight is cheaper: - IDR X
If same price: + IDR 0

Where X = abs(previously_chosen_total - available_total) / number_of_passengers (including children & infants)
```

For round trip: departure and return priced independently against their respective pre-defined legs.

Pre-selected flight pinned at top of list. User does NOT choose fare here — Order auto-selects cheapest. After selecting both legs (round trip), user returns to Bundling SRP.

Errors → redirect to **Flight x Hotel SRP** (not standalone Flight error page).

**B2B Corporate**: Non-compliant fare → popup that corporate admin approval may be required later.

### 5.5 Flight PRP (Existing, with Tweaks)

When user clicks "Change benefit" from bundling SRP:

- Shows fare list with price scheme (upgrade delta per pax)
- Pre-selected fare has selected state
- **Hidden fares**: 100% refund/reschedule, free seat selection, free meal selection
- Fare upgrade per pax calculation example: 3 adults + 1 child + 1 infant, upgrade = IDR 300K/adult → displayed as `+ IDR {{300,000 × 3 / 5}}`
- Round trip separate: depart & return steppers shown; user selects both before returning to bundling SRP
- ~~"See detail" bottom sheet for refund/reschedule/price~~ — removed from bundling PRP flow

### 5.6 Hotel Room List & PDP (Existing, Reused)

**Disabled features**: Change date, rooms, guests, change button, tiket points earning display, wishlist.

**Sorting**: Bundling rate (biggest gap) → other rate (cheapest). Pay-at-hotel excluded.

**Constraint**: User cannot choose more than one room type (enforced in FE Bundling).

**Validation**: Existing Accommodation validations (room selection → booking form) are NOT applied in bundling flow. All validation deferred to Booking Form → Payment Page transition.

**Price display**: Same Flight + Hotel formula; round up when division is uneven.

### 5.7 Booking Form

**Summary section**: Single card with Flight and Hotel subsections + Baggage & Allowance Policies link.

**Contact details form**: Salutation, name, phone, email. Smart Profile integration. Member data autofill. Nationality field (used for vendor issuance — already accommodated).

**Passenger/guest details**: One form per flight passenger. Hotel guest = subset of flight passengers. Each room needs one adult representative for check-in. Each adult can only represent one room.

**Hotel special requests**: Checkbox + free text, forwarded to Accommodation.

**Price detail footer**: Combined bundled price only. Fees and taxes combined (not separated by vertical). No per-item breakdown.

**Back button behavior**: User chooses between returning to Bundling SRP (reset pre-defined flight + hotel on top of list) or staying in form.

**Member flows**: Login banner, Smart Profile, new profile during order creation, profile autofill, UNM validation (links to MEM Confluence pages).

**B2B Corporate flow**: Travel policy evaluation → possible redirect to B2B Booking Form with preserved packaged price in smallest currency unit. Corporate interim page for approval flow. Policy failure → error + back to form. IDR-only constraint: non-IDR → blocking popup.

### 5.8 Payment Page

- Order summary for both items
- Payment time limit: shorter of flight or hotel limits
- Phase 1: Normal payment methods only (all currently available methods)
- Pay at Hotel disabled
- No per-item price breakdown; bundled price only
- Fees/taxes shown aggregated for flight and hotel (not split per item)
- Tech solutioning must be scalable for down payment and flexible payment (Phase 2)
- B2B Corporate: Travel policy validation with multi-vertical data; IDR-only gate — non-IDR → blocking popup

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
1. **Inventory unavailable** (disruptive):
   - Hotel unavailable → redirect to Bundling SRP with prior flight as pre-defined
   - Flight unavailable → redirect to Bundling SRP with chosen hotel on top
2. **Price increase** (informational) — popup, user can proceed
3. **Price decrease** — no popup, silently applied
4. **Benefit change** — TBD
5. **Booking timeout** — expiry handling
6. **General error** — retry/redirect

Multiple errors are **stacked** in order of occurrence.

Bundling maps **inventory** errors and **price change** errors to dedicated bundled error states. Other rate-plan errors follow vertical-specific patterns.

### 6.3 SEO & URL Slugs

SEO copy provided in PRD for both ID and EN:
- H1, meta title, meta description defined
- Slug format under discussion (e.g., `/flight-hotel/`, `/bundling/`, `/package/`)
- New SEO templates and sitemaps required
- SEO service owned by Order team for bundling pages

### 6.4 Inventory & Kuber Headers

- Order calls Flight and Accommodation for inventory; each returns filtered/sorted lists; fallbacks owned by each vertical
- Kuber/pricing headers must match assessment document (linked from PRD)
- Flight passes pre-defined data: total price, airline, seat class, search flag for bundling
- `isBundling` flag from Bundling service; same flag used for Kuber evaluation on FE
- Additional API parameter on **all** bundling calls to mark origin as bundling

### 6.5 Validation Flow (Booking Form → Payment)

No separate "revalidate itinerary" step for initial bundling scope. Flight + Hotel validated **together before order creation** as a single validation pass:

1. **Room representative validation**: Each room must have one adult; each adult can only represent one room
2. **Inventory check**: Order service validates with both Flight and Accommodation services
3. **Price revalidation**: Comparison of current vs. original prices
4. **Flight churning / non-hold**: Handled inside flight vertical; bundling only needs success/failure signal

---

## 7. Price Calculation Logic

### 7.1 Core Formula

```
Per-pax bundled price = (Flight total price for ALL passengers + Hotel total price for ALL rooms & ALL nights) / Number of travelers (including children and infants)
```

Round up when division is uneven.

### 7.2 Price in Flight PRP Fare Card (Entry Point)

```
Bundled price per pax = (Flight cheapest fare total + Hotel first-in-list total) / Number of travelers
Booked-separately price = Bundled price × 125 / 100
```

The "booked-separately" price is a synthetic comparison price (25% markup). Slashed price display: **TBC** — 125% is mentioned but not finalized.

### 7.3 Price Scheme in Flight SRP (Change Flight)

```
Delta = (Available flight total - Previously chosen flight total) / Number of passengers (including children & infants)
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
- Each vertical calls Kuber independently with additional `is_bundling` flag and extra params (room count, etc.)
- Phase 1: Existing vertical budget pockets (confirmed with corpstrat/commercial — no separate bundle budget)
- Phase 2 consideration: Orchestrator-level Kuber call with bundle vertical ID spanning multiple verticals; budget split TBD

### 7.6 Tax and Fees

- Displayed as a **combined** figure for both flight and hotel
- Not separated per vertical in any user-facing context
- Price filter on hotel cards: **after tax** (aligned with Figma)
- Convenience fee: stored per vertical in their respective cart/collection tables; not in Order DB
- Payment-team additional fees: stored as consolidated field; allocation depends on whether evaluation is at order_id or order_detail_id level

---

## 8. Issuance, Error Handling & Order Lifecycle

### 8.1 Issuance Sequence

```
User pays → Order service triggers Flight issuance
  ├── Flight issuance FAILS → Auto-refund BOTH flight AND hotel → Both transition to BU status
  └── Flight issuance SUCCEEDS → Order service triggers Hotel issuance
        ├── Hotel issuance FAILS → Partial refund for hotel ONLY (auto-refund after 3 retries for B2C)
        │   Flight remains issued; failure info shown on accommodation card
        └── Hotel issuance SUCCEEDS → Order status updated, purchase trackers fired with bundling origin flag
```

### 8.2 Key Issuance Decisions

- **No hold-and-resume**: ~~Hotel will hold booking using Expedia hold API~~ — struck through in PRD. Hotel issuance proceeds directly after flight success. Confirmed by Accommodation PM: should NOT be limited to Expedia inventory.
- **Sequential, not parallel**: Flight MUST succeed before hotel issuance begins.
- **Partial issuance is possible**: Flight success + hotel failure = partially issued order.
- **No flight revalidate itinerary**: For initial bundling scope, no separate flight revalidation step. Order provides combined validation flow.
- **BU workflow**: Per order_detail_id level.

### 8.3 Validation Flow

All validations are collated into a **single validation pass** at the Booking Form → Payment Page transition:

1. **Room representative validation**: Each room must have one adult; each adult can only represent one room
2. **Inventory check**: Order service validates with both Flight and Accommodation services
   - Hotel unavailable → redirect to SRP (flight preserved as pre-defined)
   - Flight unavailable → redirect to SRP (hotel on top of list)
3. **Price revalidation**: Comparison of current vs. original prices
   - Total increases → informational popup
   - Total decreases → silent pass-through
4. **Flight churning / non-hold**: Handled inside flight vertical; bundling only needs success/failure

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
- User can refund hotel while keeping flight, or vice versa — **no** requirement to refund/reschedule both when one item is refunded
- Refund amount for hotel uses the opaque amount (not the normal rate)
- No "bundle-level" single refund — each item refunded independently
- Confirmed in QnA: "refund will be done per order detail id level where accommodation order detail and flight order detail will preserve exactly like current Smart trip refund"

### 9.2 Reschedule

- Follows per-vertical existing policy at L2 level
- Independent per item
- Hotel SSRR (Self-Service Rebooking/Refund) still possible for B2B Corporate; hotel price shows delta vs. previous booking

### 9.3 Post-Purchase Insurance (PPI)

- Enabled per L2 item after order is fully paid
- Creates separate L2 order, linked as related order (on flight order)
- Receipt on PPI order only
- Existing implementation; no bundle-specific changes

### 9.4 Post-Purchase Add-Ons (PPA)

- Same as PPI: enabled per L2 item, separate L2 order, existing implementation
- Receipt on PPA order only
- **Note from Accommodation PM**: Pre-purchase add-ons are NOT available in Phase 1 booking form

### 9.5 Loyalty / Tiket Points

- Earn rate: `(BASE_PRICE + PROMO_CODE + GIFT_VOUCHER + TIX_POINT) × 1%`
- Combined from both verticals (each vertical calculates; bundling displays what verticals provide)
- If one item fails issuance, points earned only for successfully issued item
- Instant Point Cashback (IPC) revocation on partial refund: revoked at **order_id level** (all points revoked, not proportional to refunded item). If points already used, deducted from refund amount.
- Tiket points earning display is suppressed in SRP/PDP during Phase 1
- Loyalty scope for point earning aligned with corpstrat answer (pending)

---

## 10. Platform Integration

### 10.1 My Order

- L1 bundling order: single card when **unpaid**
- L2 items: separate cards when **paid**, following existing vertical display
- Pre-payment cancellation follows existing flow
- Order title and icon mapped internally (TBC)
- `isBundling` flag same field as Kuber

### 10.2 E-Receipt

- **Single receipt** covering both flight and hotel
- Product name: "Flight + Hotel Bundling"
- Contains: Order ID, order name, email, phone, paid date, payment method, description (per-item details allowed as long as no per-item price breakdown), product count, subtotal, admin fee, total payment
- Same receipt displayed in both Flight and Hotel L2 order details

### 10.3 E-Voucher

- Accessed per L2 order detail (flight voucher in flight order, hotel voucher in hotel order)
- Follows existing implementation per vertical
- Order handles flight e-voucher errors; Accommodation handles hotel e-voucher errors

### 10.4 Notifications (WA & Email)

- Receipt WA/email per Payment team flow
- Issuance notifications follow **sequential** order: flight first, then hotel
- Three scenarios: both issued, only flight issued (hotel failed), both failed
- Failure notifications use **vertical** failed-issuance notifications if implemented
- E-voucher attachments follow vertical email flows
- No per-item price breakdown in notifications; bundled total + combined taxes

### 10.5 Promo

- Bundling flag added to `dynamicPromo` object
- Promo system validates to **order_detail_id** level (one order_id, multiple order details / orderTypes)
- Promo system consumes both `orderType` and `productType` (from Payment) — needs logic adjustment
- Promo creation: must select both flight AND hotel verticals; new criterion in dynamic promo
- All current promo functionality (including promo detail page) preserved for bundling
- Items flagged as **bought in bundle** for reporting

### 10.6 Payment

- Vertical recognition: via `paymentUnitDetail.productType[0]` — first element of payment unit detail product type array
- Not creating a brand-new vertical at payment level; preserving existing vertical identifiers at order detail level
- Payment team needs to adjust how they identify bundling orders
- Bundled total at L1 without per-vertical line items in the payable amount; L2 totals sent internally

### 10.7 Financial Dashboard / CFD

- Each bundling transaction recorded per vertical following existing flow
- Additional flag differentiates bundle vs. standalone purchase
- No separate price components owned by bundling team — all sourced from verticals and/or payment
- New product category at L1 level for bundling
- Sales report: each vertical's reporting + flag for separate vs. bundle purchase

### 10.8 Admin Care (CS Dashboard)

- **Order list**: Search by order ID shows L2 items (flight card + hotel card) with standard columns (order id, detail id, dates, product type, passenger, invoice, discount, paid amount, status, etc.)
- Each item displays same info and actions as standalone orders
- **Related orders** section links flight and hotel items within the same bundle (example: RT flights + hotel cross-linked)
- Invoice and paid amount covers full bundling amount (from Payment service)
- Payment history from Payment team (all verticals)
- Receipt and e-voucher modals same as consumer-facing
- CS sees same receipt; Order handles flight e-voucher errors, Accommodation handles hotel e-voucher errors

### 10.9 Data / BI

- Bundling data accessible via BigQuery with same flows + bundling flag
- Order counting: `order_id` represents one checkout; vertical-level `count(distinct order_id)` still works per vertical, but cross-vertical aggregation shows 1 order; provide multi-vertical (bundle) identifier in data
- Promo, GV, subsidy split: discussion with commercial/corpstrat scheduled (pending)
- Fare structure: no difference if data captured at order_detail_id level; if at order_id level, need agreed split/allocate

### 10.10 Insurance

- Phase 1: 100% refund/reschedule insurance **disabled** as flight fare option in bundles
- Phase 1: Insurance excluded from booking form
- Phase 2: Insurance as add-on in booking form
- Flight team needs parameter to differentiate bundle vs. standalone for insurance logic
- VI handling: follow-up needed on insurance selected in background with flight purchase

### 10.11 Discovery / Homepage

**Phase 1 scope (confirmed with Discovery PM):**

| Module | Status | Notes |
|--------|--------|-------|
| Vertical Icon | No dev, dashboard config only | Request via Google Form, align with #homapagecurator Slack |
| Global Search (keyword redirect) | Needs development | Keywords TBD |
| Global Search (query in search) | Needs development | |
| Upcoming Booking | Include for Phase 1 | Per order detail; test during integration |
| Continue Payment | Include for Phase 1 | Uses existing `/tix-order-composer/yourorder/v1/waitingpaymentlist` — no API changes; data treats F+H as one vertical order |
| Review Widget | Include for Phase 1 | Hotel reviews from bundling bookings; test in integration |
| Promo List | Include for Phase 1 | Order PM creates new Flight+Hotel promo category; no dev, need testing |
| Page modules (Inventory, Contextual) | **Out of scope Phase 1** | Deferred per discussion with Discovery PM |
| Cross-Sell GV | **Out of scope Phase 1** | |
| Recent Search | **Out of scope Phase 1** | |

### 10.12 B2B Corporate Dashboard

- Purchase report: per-vertical with bundle vs. separate flag
- Corporate management fee: currently at mixed granularity (e.g., flight per pax); needs adjustment for bundling
- Travel policy enforcement throughout the flow (SRP, booking form, payment)
- IDR-only constraint enforced with blocking popup for non-IDR currencies

### 10.13 Member

- Login banner, Smart Profile, new profile during order creation, profile autofill, UNM validation
- No bundling-specific development needed
- Guest account creation and optional email flows supported

### 10.14 SEO

- New SEO templates and sitemaps for bundling pages
- SEO copy provided in ID and EN (H1, meta title, meta description)
- URL slug format under discussion
- SEO service owned by Order/Bundling team

---

## 11. Dependencies Matrix

| Team | Dependent? | Key Deliverables | Development Required? |
|------|-----------|------------------|----------------------|
| **Accommodation** | **Yes** | Entry point banner, bundling rate inventory, search/filter/sort, issuance flow, fallback handling, promo objects, post-purchase flag | Yes |
| **Flight** | **Yes** | Entry point (LP banner + PRP fare card), inventory with price scheme, PRP fare tweaks, differentiate bundle vs standalone, promo objects, post-purchase flag | Yes |
| **Payment** | **Yes** | Bundling order recognition via productType, receipt generation, payment history API | Yes |
| **Promo** | **Yes** | `dynamicPromo` flag support, new criterion for bundling, orderType+productType logic adjustment. **Update:** needs development even if vertical name remains (validate to order_detail_id level) | Yes |
| **Insurance** | **Yes (Phase 1 partial)** | Disable 100% R/RR fare for bundles; Phase 2: booking form insurance. Flight to pass bundle flag. | Yes (Flight to pass bundle flag) |
| **Loyalty** | **Yes** | Point earning calculation (combined verticals), IPC revocation handling | Yes (depends on corpstrat answer) |
| **Discovery** | **Yes** | Vertical icon config, global search, promo list, upcoming booking, continue payment, review widget | Partial (global search query + testing) |
| **B2B Corporate** | **Yes (QA + travel policy)** | Travel policy validation, IDR gating, corporate booking form integration, management fee adjustment | Yes (testing + policy flow) |
| **SEO** | **Yes** | New templates, sitemaps, URL slugs for bundling pages | Yes (bundling service by Order team) |
| **Refund** | **No (depends on solution)** | Existing per-vertical flow works | No |
| **Member** | **No** | Login banner, smart profile, profile creation, autofill, UNM validation | No |
| **Pricing** | **No** | Kuber evaluation with bundling flag, deferred cost strategy | No |
| **CFD** | **Yes (QA only)** | Same flow + bundling flag | No (QA testing) |
| **Data** | **No (depends on solution)** | BQ access with bundling flag | No |
| **B2B Affiliate** | **No** | No plan for API channel bundling; possibly offline tour agent in 2027 | No |
| **TTD** | **No** | Review and feedback only (Phase 3 inclusion) | No |
| **NFT (AT)** | **No** | Review and feedback only (Phase 3 inclusion) | No |
| **Growth** | **No** | Cross-sell recommendation (future) | No |
| **CS & Help Center** | **No** | Knowledge sharing, help center content | No |
| **Commercial** | **No** | Budget pocket alignment, promo/GV/subsidy split decisions | No |

---

## 12. Key Decisions & Clarifications from Cross-Team QnA

> Full QnA log maintained on the [Cross Team QnA](https://borobudur.atlassian.net/wiki/spaces/PURE/pages/4565336086) child page. Key decisions summarized below.

### 12.1 Order Structure

> **Q**: Will bundling introduce a completely new order data structure?
> **A**: No. The existing structure already supports multiple `order_detail_id` (L2) under one `order_id` (L1). Examples: train+insurance, flight+insurance. No backward-incompatible changes. Order id is the transaction/checkout entity; vertical/type lives at order detail via `item_type`.

### 12.2 Refund Granularity

> **Q**: Will refund be per-bundle or per-item?
> **A**: Per order_detail_id level. Flight and hotel refunds operate independently, exactly like current smart-trip refund behavior. This will hold until multi-city (segment segregation) changes behavior.

### 12.3 Pay at Hotel

> **Q**: Is Pay at Hotel applicable for packaged rates?
> **A**: Rate key (package, basic, etc.) is separate from payment method (pay now, PAH). Only one vendor supports PAH (Booking.com). Even if a vendor returns PAH for a package rate, it will be filtered out. PAH is out of scope for bundling.

### 12.4 Hotel Search Area Restriction

> **Q**: Can accommodation search results include nearby areas?
> **A**: Accommodation uses "Lepus" (lat/long within 10km radius) for POI/near-me searches and "Fornex" (atlas ID) for city-level searches. POI is generalized to city level for bundling. User can search POI (e.g., airport) or City. If Accom can map POI to broader area, both supported; else Atlas polygon at city level then search.

### 12.5 Guest Names for Hotel

> **Q**: Should every flight passenger be sent to accommodation provider?
> **A**: One PIC (person in charge) per room is sufficient. PIC should be selected from the passenger list. Exact guest name requirements depend on each vendor.

### 12.6 B2B Affiliate

> **Q**: Will B2B Affiliate adopt bundling?
> **A**: No plan for API channel bundling. Bundling uncommon in API partnerships. Possibly offline tour agent in 2027. B2B partners have stricter SLAs; solution should be separate from B2C. Acknowledged by Order PM.

### 12.7 Subsidy Budget

> **Q**: Will bundle subsidies use existing vertical budgets or a new bundle budget?
> **A**: Existing vertical budget pockets (agreed with corpstrat and commercial). No dedicated bundle budget for Phase 1.

### 12.8 Convenience Fee

> **Q**: How will convenience fee be handled for multi-vertical orders?
> **A**: Vertical convenience fee is stored in each vertical's cart/collection table. For additional fees from Payment team, stored as consolidated field. Allocation depends on whether evaluation is at order_id or order_detail_id level.

### 12.9 Insurance Linkage

> **Q**: Insurance is currently linked at order_id level. How to identify which vertical it belongs to?
> **A**: Check with Insurance PM whether each insurance product (credential) is bound to a vertical entity. If bundling is one order_id with multiple verticals, insurance should point to the right vertical. **Pending Insurance PM response.**

### 12.10 Promo/GV/Subsidy Split

> **Q**: Promo, GV, and subsidy are at order_id level. How to split across verticals?
> **A**: Discussion scheduled with commercial, corpstrat, and data team. Follow-up session completed. **Split rules still TBD.**

### 12.11 Loyalty IPC Revocation

> **Q**: How does IPC work on partial refund?
> **A**: Points revoked at order_id level (all points revoked regardless of which item is refunded). If points already used, deducted from refund amount.

### 12.12 B2B Corporate Travel Policy

> **Q**: How are travel policy checks handled?
> **A**: Flight and Hotel services return inventory based on B2B corp user's account properties. No additional bundling-specific development needed for travel policy. Pages: SRP flight (reused), PDP flight (Flight PRP), booking form flight; SRP hotel (new bundling SRP with predefined flight), PDP hotel (reused), booking form hotel.

### 12.13 Sequential Issuance & Hotel Failure

> **Q**: What if flight is issued but hotel issuance fails?
> **A**: Hotel implements auto-refund for B2C after 3 retry failures. Same failure information shown in accommodation card. The opaque hotel amount is refunded. Flight fail → both flight and accom → BU then auto refund. Both ok → fully issued. Hotel SSRR for corp still possible.

### 12.14 SEO & Slugs

> **Q**: Will bundling need new SEO templates and sitemaps?
> **A**: Discussion session created; slug format TBD (e.g., `/flight-hotel/`, `/bundling/`, `/package/`). **Unresolved.**

### 12.15 Opaque Pricing for Corporate

> **Q**: How to prioritize bundling opaque price for B2B Corporate?
> **A**: Only Expedia (1 of 9 aggregators) provides opaque price today. Cheaper buying price only when bundled. Other inventory: Accom team controls which rate key is sold in F+H flow.

### 12.16 Insurance in Bundle

> **Q**: How should insurance work in the bundling flow?
> **A**: Exclude insurance in booking form; turn off 100% refund for bundle product; Flight helps pass param (flight-only vs bundle). Follow-up: Order PM × Flight PM on VI handling (insurance selected in background with flight purchase).

### 12.17 Product Bundling by User Type

> **Q**: Is bundling limited to specific user types?
> **A**: No exception. Bundling available for **all** B2C, B2B Corporate, and metasearch users.

### 12.18 Booking Form Construction / Passenger

> **Q**: How are booking form fields constructed for multi-vertical?
> **A**: Union of flight + hotel booking forms. Booker as today. Flight pax explicit. Hotel guests ⊆ flight pax. Same profile IDs per account (unless IDs are per-vertical unique).

### 12.19 B2B Corporate Partial Issuance

> **Q**: How does partial issuance work for corporate orders?
> **A**: Same sequential issuance: Flight → Hotel. Flight fail → hotel not issued (fully failed). Flight ok, hotel fail → partial failure. Both ok → fully issued. Hotel SSRR for corp still possible; hotel price shows delta vs. previous booking. Refund/reschedule partial at order detail level; policies per vertical. **Open**: Who absorbs price delta (Tiket vs user) on hotel rebooking.

### 12.20 Accommodation Funnel Errors

> **Q**: How are accommodation errors handled in bundling flow?
> **A**: Errors combined on Order creation (path to payment). Price change → user notified, can still proceed. Unavailability → message + redirect to bundling SRP.

### 12.21 Room List: Filters, Sorting, PAH

> **Q**: What are the room list constraints for bundling?
> **A**: PAH inventory excluded. Room fare rules TBD (subject to discussion). Inventory priority: cheapest → bundling rate. Sorting: bundling rate (biggest rate gap) → other rates (cheapest). User cannot pick more than one room type (enforced by FE Bundling). NHA eligible. Refundable-only via syspar (configurable for bundling side).

### 12.22 ML/Sorting for Hotels

> **Q**: How does ML interact with bundling hotel sorting?
> **A**: No new ML for Phase 1. Sort applied **after** existing ML ranking. If sorting by discount price, ML not used — only price engine. Default + manual filter. Parallel pipeline: get recommendation + get price. Future phase: new ML with bundling discount + airline as parameters. Hotel ads relevance still being evaluated.

### 12.23 Price Display: Before or After Tax

> **Q**: Should hotel card prices show before or after tax?
> **A**: After tax (aligned with Figma).

### 12.24 "Hotel Fully Booked" State

> **Q**: Will the "hotel fully booked" card state be used?
> **A**: That scenario is impossible — sold-out hotels are not returned by Accommodation service; inventory not updated unless user scrolls.

### 12.25 Nationality on Booking Form

> **Q**: Is nationality field needed for hotel booking?
> **A**: Already accommodated. Nationality is used for vendor issuance.

### 12.26 Discovery Page Modules

> **Q**: What Discovery modules are in scope for Phase 1?
> **A**: Vertical icon (config only), global search, upcoming booking (per order detail), continue payment (per order level, no API changes), review (hotel only, test in integration), promo list (F+H category). Page modules (inventory, contextual), cross-sell GV, recent search — all out of scope Phase 1.

### 12.27 Member Functionality

> **Q**: What member features are needed at initial release?
> **A**: Login banner, Smart Profile, new profile during order creation, profile autofill, UNM validation.

### 12.28 Order Counting for BI

> **Q**: How should bundling orders be counted per vertical vs. total?
> **A**: Adjust counting: order_id wraps multi-vertical checkout. Provide multi-vertical (bundle) identifier in data. Vertical-level counts still possible. Checkout-level = one order with bundle flag.

### 12.29 B2B Commission

> **Q**: How does B2B commission work with bundling?
> **A**: Commission as today per vertical providing the service. B2B API not adopting bundling in Phase 1; B2B Non-API possibly later.

### 12.30 Corporate Management Fee

> **Q**: How is management fee handled on top of vertical/payment fee?
> **A**: Currently at mixed granularity (e.g., flight per pax). Needs adjustment for bundling.

### 12.31 Add-ons, Meals/Seat, TixPoints in Booking Form

> **Q**: How are add-ons and tiket points handled?
> **A**: Phase 1 excluded. Phase 2+ discussion for multi-fare with free add-ons requiring pre-booking selection. TixPoints: each vertical calculates; bundling displays what verticals provide. Display: total F+H; pre-booking split to pax level; booking & payment show total; tax+fee as one payment entity.

### 12.32 B2B Corporate SRP & PDP Pages

> **Q**: Which pages are reused vs. new for B2B Corporate?
> **A**: SRP flight: reused. PDP flight: Flight PRP. Booking form flight: reused. SRP hotel: new (bundling SRP with predefined flight). PDP hotel: reused. Booking form hotel: reused.

---

## 13. Open Questions & Unresolved Items

### 13.1 From PRD (marked TBC/TBD)

| Item | Context | Status |
|------|---------|--------|
| Slashed price formula in SRP | What is the "booked separately" comparison price? | TBC — 125% of bundle price is mentioned but not finalized |
| Price detail in fare bottom sheet | What price components shown when user clicks "see detail" in PRP fare card | Partially addressed — bottom sheet ~~removed~~ from bundling PRP; total F+H + tax/fee as one entity |
| Children handling from Flight LP deeplink | If children/infants come from flight LP, should bundling LP force age input before proceeding? | Open — Hotel Search Details Modal addresses this (default ages: child 11, infant 1) but inline comment unresolved |
| Flight chosen preservation on change | When user changes flight in SRP, should the previous flight selection be preserved for back-navigation? | Open |
| Notification content | Exact WA/email notification copy for success, partial failure, full failure | TBC |
| Order title and icon mapping | How bundling orders appear in My Order list | TBC |
| Benefit change error handling | What to display when benefit/fare changes | TBD |

### 13.2 From Cross-Team QnA (Still Open)

| Item | Context | Status |
|------|---------|--------|
| Promo/GV/Subsidy split rules | How to split order-level promo, GV, subsidy across verticals | Discussion completed, split rules still TBD |
| SEO slug format | `/flight-hotel/`, `/bundling/`, `/package/` — which? | Discussion session created, unresolved |
| Insurance linkage to vertical | Insurance at order_id level; which vertical does it belong to? | Pending Insurance PM response |
| Who absorbs hotel rebooking price delta | When SSRR results in price change, Tiket or user pays delta? | Open (from B2B Corporate QnA) |
| Corporate management fee for bundling | Mixed granularity needs adjustment | Open |
| Loyalty scope / corpstrat alignment | Point earning rules need corpstrat answer | Pending |
| Hotel ads in bundling SRP | How hotel ads interact with bundling sort | Under evaluation |
| VI handling for insurance | Insurance selected in background with VI flight purchase | Follow-up needed (Order PM × Flight PM) |
| Receipt / e-ticket format | Single receipt vs. separate e-tickets for bundled purchases | Open |
| Opaque leakage on flight refund | If flight refunds and hotel still issued, opaque price exposed | Open |
| Multi-room packaged rate | How multi-room works with package rates | Open |
| One-way package feasibility | Single-trip bundling with accommodation | Open |
| Multi-city / open-jaw hotel eligibility | Which hotels eligible when flight has disconnected legs | Open (Phase 2 prerequisite: multi-city flight) |
| Involuntary flight reschedule repricing | How package price recalculates on involuntary changes | Open |
| Cancellation alignment flight vs. hotel | Partial pax cancellation scenarios | Open |
| Cart / OMS architecture | Cart orchestration, BFF, source-of-truth design | Open |
| Refund UI related orders | Should related bundle items show when user requests refund on one item? | Open |
| Room fare rules | Specific room-level fare rules for bundling | TBD (subject to discussion) |
| Attribution / UTM | F+H attributed to one utmSource or split? | Discussion session created |

### 13.3 From Confluence Inline Comments (Unresolved)

| Comment | Author Context | About |
|---------|---------------|-------|
| "apakah chosen flight(s) bisa dipreserved dan dijadikan sebagai pre-defined flight" | FE engineer | Flight preservation when coming from landing page with pre-selected flight context |
| "confirm to pm accom the logic" on room count calculation | PM | Whether accommodation's existing room-count logic matches bundling's `ceil(adults/2)` rule |
| "nggak ngerti examplenya" on first flight changes pricing | Flight PM | Clarity needed on the price scheme example in the PRD |
| "ini ga consider non-direct properties lain ya?" on cheapest non-direct selection | Flight PM | Should pre-defined flight consider transit count and transit time, not just price? |
| "no, in SRP flight currently we only show price per pax based on the adult price" | Flight PM | Potential clash between flight's adult-only per-pax display and bundling's all-pax display |
| "i believe we still cannot purchase post purchase adds on" | Accom PM | Post-purchase add-ons typically happen before payment; clarification needed on post-payment add-on flow |
| Price calculation for hotel tax/fee | FE engineer | Whose responsibility is tax aggregation: Accommodation or Order service? |
| Issuance confirmation flow | FE engineer | Is it Flight→Hotel direct or Flight→Order→Hotel? |
| "related orders" in refund page | Refund PM | Should related bundle items show when user requests refund on one item? |

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
| **NHA** | Non-Hotel Accommodation — property types like villas, apartments, hostels. Eligible for bundling. |
| **CFD** | Centralized Finance Dashboard — financial ledger and reconciliation system. |
| **Syspar** | System parameter — server-side configuration that can be changed without deployment. |
| **Fornex** | Accommodation's ML-based search engine using atlas IDs for city-level and above searches. |
| **Lepus** | Accommodation's proximity-based search engine using lat/long within 10km radius. |
| **Atlas ID** | Geographic identifier used by Accommodation search (POI generalized to city level for bundling). |
| **Coach Mark** | First-time user UI highlight/tooltip that calls attention to specific features (Change benefit, Change date). |
| **Rate Key** | Accommodation pricing identifier (package, basic, etc.) — separate from payment method. |
| **isBundling** | Boolean flag passed across services and to Kuber to identify bundling context. |
| **dynamicPromo** | Promo system object that receives bundling flag for promo eligibility and creation. |

---

## 15. Reference Documents

| Document | Location | Description |
|----------|----------|-------------|
| **Main PRD** | [Confluence PURE/4416700417](https://borobudur.atlassian.net/wiki/spaces/PURE/pages/4416700417) | Full PRD with requirements tables and acceptance criteria |
| **Cross Team QnA** | [Confluence PURE/4565336086](https://borobudur.atlassian.net/wiki/spaces/PURE/pages/4565336086) | Running Q&A log with all cross-team discussions |
| **Price Aggregation Documentation** | [Confluence PURE/4744970410](https://borobudur.atlassian.net/wiki/spaces/PURE/pages/4744970410) | Price simulation spreadsheet + data mapping diagrams (bundling, accommodation, flight) |
| **Error Message in Booking Process** | Confluence (draft) | Error message documentation for booking flow (in progress) |
| **Price Simulation Spreadsheet** | [Google Sheets](https://docs.google.com/spreadsheets/d/1qL7DB97Fdkhagu5I_hVWxaxkjNIM-o7IdvjBBMR_4cA/edit?usp=sharing) | Price calculation examples and scenarios |
| **Bundling Data Mapping** | [draw.io diagram](https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1&dark=auto#G1CZGUhBAleWQn3gRcM9fN2knTD2RfZ61P) | Visual data flow mapping for bundling |
| **Admin Care Design** | Figma (linked from PRD) | CS dashboard design for bundling orders |
| **B2C Customer E2E Flow** | Figma (linked from PRD) | Full customer journey design |
| **UI Tracker** | Google Slides (linked from PRD) | UI status tracking across pages |

---

> **For AI agents reading this document**: This spec represents the Hotel x Flight Bundling initiative for Tiket.com's Order Platform (Confluence version 99, last synced 2026-04-16). The Order/Bundling team acts as an orchestration layer over existing Flight and Accommodation microservices. Key invariants: (1) opaque combined pricing with no per-item breakdown to users, (2) sequential Flight→Hotel issuance, (3) L2 items treated as independent regular orders post-payment, (4) existing vertical services retain full domain ownership, (5) no new order data structure — existing L1/L2 hierarchy is reused. When analyzing this initiative's impact on the Order Platform architecture (status tracking, price breakdown, financial reconciliation), note that bundling introduces a new L1 order type but does NOT change the fundamental L2 data model or vertical service boundaries. B2B Corporate is in scope for Phase 1 with travel policy enforcement and IDR-only constraints.
