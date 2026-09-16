# Guest Growth Domain

## Domain Overview

Guest Growth owns the **guest-facing digital experience** beyond ticketing: the Companion App PWA, virtual queuing (Spot Saver), personalized recommendations, nudge-based engagement, and return-visit optimization. The goal is to grow the estate from 5,000 to 15,000 daily visitors — and keep them coming back.

**Key Invariant:** The recommendation engine never sends a guest into a congested zone. Every nudge is checked against the current crowd simulation before delivery.

---

## Module Decomposition

| Module | Responsibility | AI Involvement |
|---|---|---|
| **Companion App Backend** | API for PWA: ticket wallet, maps, wait times, stories | None (serves data) |
| **Spot Saver** | Unified virtual queue orchestration for rides + enclosures | None (event coordination) |
| **Guest Profile** | Consent management, preferences, visit history, segments | None |
| **Recommendation Engine** | GBDT ranking model + GenAI copy drafting for suggestions | ML ranking + GenAI |
| **Nudge Engine** | Push, email, in-app message orchestration | GenAI copy drafting |
| **Churn Prediction** | Return propensity scoring [0, 1] | ML classification |
| **Campaign Manager** | A/B testing, attribution, experiment management | None (deterministic) |

---

## Companion App — Offline-First PWA

The companion app is a **Progressive Web App** (PWA) that works without installation and degrades gracefully offline:

| Feature | Online | Offline |
|---|---|---|
| **Ticket Wallet** | Live QR with real-time validation | Cached QR (Ed25519-signed, self-validating) |
| **Estate Map** | Live crowd heatmap overlay | Bundled vector map, static paths |
| **Wait Times** | Real-time from Park Ops events | Last-known values (stale indicator) |
| **Recommendations** | Personalized GBDT-ranked list | Proximity-based sorting from BLE |
| **Stories** | Live GenAI-narrated enclosure stories | Cached last-fetch stories |
| **Spot Saver** | Full queue management | View existing reservations only |
| **Notifications** | Real-time push | Queued for delivery on reconnect |

### Technology Stack

| Component | Choice | Rationale |
|---|---|---|
| Framework | PWA with service workers | No app store dependency; works for international visitors |
| Local storage | LiteDB (via IndexedDB) | Structured offline storage for tickets, preferences, history |
| Maps | Bundled vector tiles | No tile server dependency offline |
| Proximity | BLE beacon scanning | Zone detection without WiFi; works with phone in pocket |
| Sync | Background Sync API | Automatic foreground data sync on reconnect |

---

## Spot Saver — Unified Virtual Queue & Reservation Engine

Spot Saver is one guest-facing queue for two very different kinds of capacity: heritage mechanical rides, whose throughput is governed by Park Operations, and animal exhibits, whose occupancy is governed by Living Collection's welfare limits. Guest Growth owns the experience and neither ledger.

For **exhibits**, `VisitRequested.v1` $\to$ `VisitConfirmed.v1` is the **single reservation path** across the estate — the same two events carry a slot booked weeks ahead at checkout (`origin: pre_visit`) and a slot booked from a bench ten minutes before the feed (`origin: in_park`), and both draw down the same welfare-clamped cap. For **rides**, Guest Growth holds the virtual queue position itself and publishes `QueueJoined.v1` / `QueueCalled.v1` / `QueueExpired.v1` so Park Operations can balance dispatch against the queue it can no longer see on the ground:

```mermaid
sequenceDiagram
    autonumber
    participant Guest as Guest (Companion App)
    participant GG as Guest Growth (Spot Saver Facade)
    participant LC as Living Collection (Capacity Authority)
    participant PO as Park Operations (Wait Times)
    participant Kafka as Kafka Event Backbone

    Guest->>GG: "Reserve viewing for Piranha House at 2 PM"
    GG->>PO: Check footpath transit time (12 min from current zone)
    PO-->>GG: Transit verified
    GG->>Kafka: Publish: VisitRequested.v1 (enc-amazon-17, 14:00, party=4, origin=in_park)
    Kafka->>LC: Evaluate capacity & biological stress index
    
    alt Welfare Index OK & Capacity Available
        LC->>Kafka: Publish: VisitConfirmed.v1 (token=ss-tok-4891, window=14:00–14:30)
        Kafka-->>GG: Update local session pass
        GG-->>Guest: "Confirmed! Your QR pass is active for 14:00–14:30"
    else Capacity Full or Welfare Throttle Active
        LC->>Kafka: Publish: VisitDenied.v1 (reason=capacity_capped, next=15:00)
        Kafka-->>GG: Trigger Anti-Congestion Recommender
        GG-->>Guest: "2 PM is resting. Book 3 PM, or visit Grand Carousel (5 min wait)?"
    end
```

### UX Honesty Invariants

To maintain absolute guest trust, the Companion App enforces three non-negotiable honesty invariants:
1. **Never Under-Report Wait Times**: The app can never display an estimated wait time lower than the authoritative sensor-derived figure published by Park Operations (`RideWaitTimeUpdated.v1`).
2. **Never Route Into Closures**: A recommendation or nudge can never target an exhibit marked `closed`, `delayed`, or `quarantined` by Living Collection.
3. **Transparent Latency / Offline Indicators**: When operating offline or on stale cache, all wait times and crowd densities display an explicit "As of [time] · Offline" visual badge.

| Attraction Type | Capacity Authority | Queue Management |
|---|---|---|
| **Enclosures** | Living Collection (sole authority; GG holds no ledger) | Spot Saver reservation via GG |
| **Rides** | Park Operations (dispatch throughput) | Virtual queue position held by GG, signalled to PO |
| **Path segments** | Park Operations (advisory only) | Recommendation avoidance |

---

## Recommendation Engine

### Ranking Model

| Component | Technology | Input | Output |
|---|---|---|---|
| **Feature Extraction** | Real-time feature store | Guest profile + time remaining + crowd state + weather + party composition | Feature vector |
| **Ranking Model** | GBDT (Gradient Boosted Decision Trees) | Feature vector | Sorted attraction IDs + relevance scores |
| **Copy Drafting** | GenAI via Model Gateway | Top-3 ranked attractions + guest context | Natural language suggestion card |
| **Congestion Check** | Simulation (deterministic) | Recommended zone + current occupancy + pending recommendations | Pass/fail |

### Anti-Congestion Rule

> A recommendation is **never delivered** if it would push a zone's projected occupancy beyond the congestion threshold when aggregated with all other pending recommendations to the same zone.

This is enforced deterministically — the GenAI drafts the copy, but the simulation gates delivery.

### Recommendation Fallback Chain

| Tier | Condition | Behavior |
|---|---|---|
| **Full** | Online, model available | Personalized GBDT ranking + GenAI copy |
| **Degraded** | Model available, GenAI down | GBDT ranking + template copy |
| **Minimal** | Model down | Proximity-based sorting (nearest attractions) |
| **Offline** | No connectivity | Last-cached recommendations + BLE proximity |

---

## Nudge Engine — Retention Campaigns

```mermaid
graph LR
    subgraph Triggers["Trigger Sources"]
        Churn["Churn Prediction<br/>(return propensity < 0.3)"]
        Milestone["Visit Milestone<br/>(5th visit, birthday)"]
        Weather["Weather Event<br/>(rain → indoor suggestions)"]
        Seasonal["Seasonal Campaign<br/>(holiday, new exhibit)"]
    end

    subgraph Guard["Guardrails"]
        Consent["Consent Check"]
        Frequency["Frequency Cap<br/>(max 2/week)"]
        Congestion["Congestion Simulation"]
    end

    subgraph Delivery["Delivery Channels"]
        Push["Push Notification"]
        Email["Email"]
        InApp["In-App Message"]
    end

    Churn --> Consent
    Milestone --> Consent
    Weather --> Consent
    Seasonal --> Consent

    Consent --> Frequency
    Frequency --> Congestion
    Congestion -->|"Zone not congested"| Push
    Congestion -->|"Zone not congested"| Email
    Congestion -->|"Zone not congested"| InApp
    Congestion -->|"Zone congested"| Drop["Drop / Defer"]

    classDef trigger fill:#1e293b,stroke:#38bdf8,stroke-width:1px,color:#f8fafc
    classDef guard fill:#4c1d95,stroke:#a855f7,stroke-width:1px,color:#f8fafc
    classDef delivery fill:#064e3b,stroke:#10b981,stroke-width:1px,color:#f8fafc
    classDef drop fill:#450a0a,stroke:#ef4444,stroke-width:1px,color:#f8fafc

    class Churn,Milestone,Weather,Seasonal trigger
    class Consent,Frequency,Congestion guard
    class Push,Email,InApp delivery
    class Drop drop
```

---

## Churn Prediction Model

| Parameter | Value |
|---|---|
| **Model Type** | Gradient Boosted Classification (XGBoost) |
| **Features** | Visit frequency, recency, spend per visit, ticket type, time-of-year, weather-on-visit, party size |
| **Output** | Return propensity [0, 1] within 90 days |
| **Training Data** | Historical visit + purchase data (anonymized) |
| **Retraining** | Monthly, or when return rate diverges > 15% from predicted |
| **Evaluation** | AUC-ROC on holdout set; benchmark vs. naïve frequency predictor |

---

## Data Virtuous Cycle

```mermaid
flowchart TD
    Purchase["1. Ticket Purchase<br/>(Party, date, channel, price)"] --> Profile["2. Guest Profile<br/>(Visit history, preferences, segments)"]
    Profile --> AI["3. AI Intelligence<br/>(Demand forecast, NBA, Concierge, Churn)"]
    AI --> Experience["4. Better Experience<br/>(Personalized, frictionless, well-timed)"]
    Experience --> Spend["5. Higher Ancillary Spend<br/>(Parking, meals, timed entries)"]
    Spend --> Satisfaction["6. Visitor Satisfaction<br/>(Good experience → repeat intent)"]
    Satisfaction --> Return["7. Return Visit<br/>(More data, stronger profile)"]
    Return --> Purchase

    classDef cycle fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    class Purchase,Profile,AI,Experience,Spend,Satisfaction,Return cycle
```

---

## Published CloudEvents

All events use CloudEvents 1.0 format with ULID `id` and PascalCase `.v1`:

| Published Event (`type`) | Consumers | Key Data Fields | Architectural Impact |
|---|---|---|---|
| `QueueJoined.v1` | **Park Ops** | `queueId`, `guestSessionId`, `rideId`, `partySize`, `enrolledAt`, `estimatedReturnTime` | Park Ops balances ride dispatch & queue lane throughput. |
| `QueueCalled.v1` | **Park Ops** | `queueId`, `guestSessionId`, `rideId`, `calledAt`, `graceExpiresAt` | Park Ops summons guests to boarding platform. |
| `QueueExpired.v1` | **Park Ops** | `queueId`, `guestSessionId`, `rideId`, `reason` | Park Ops adjusts real-time ride demand and throughput. |
| `VisitRequested.v1` | Living Collection, Park Ops | `requestId`, `guestSessionId` / `orderId`, `enclosureId`, `windowStart`, `windowEnd`, `partySize`, `origin` | Exhibit reservation request with `origin: in_park`; Ticketing publishes the identical event with `origin: pre_visit`. Guest Growth never grants the slot itself. |
| `FeedbackSubmitted.v1` | Park Ops (Analytics), Ticketing | `feedbackId`, `guestSessionId`, `rating1to10`, `questionnaireAnswers{}`, `frictionSignals{}` | Estate analytics & guest satisfaction loop. |
| `ReturnVisitPropensity.v1` | Ticketing | `customerId`, `score`, `segments[]` | Ticketing personalized email/SMS offers (batch). |

---

## Failure & Degraded-Mode Matrix

| Failure | Response | Guest Experience |
|---|---|---|
| Recommendation engine down | Proximity-based sorting | "Nearby attractions" instead of "For you" |
| GenAI copy drafter down | Template-based suggestion cards | Slightly less polished text |
| Push notification provider down | Queue and retry; tickets remain valid | Delayed notification; app shows data normally |
| Living Collection events stale | Show last-known enclosure status with stale indicator | "Wait times may be outdated" badge |
| Park Operations events stale | Show last-known wait times with stale indicator | Same stale badge approach |
| Complete offline | Cached maps, tickets, last-known data | Full ticket wallet + basic navigation |

---

## Domain ADRs

- [ADR-005: Living Collection as Sole Capacity Authority](../adrs/ADR-005-living-collection-as-sole-capacity-authority.md)
- [ADR-013: Modular Monolith per Bounded Context](../adrs/ADR-013-modular-monolith-per-bounded-context.md)
