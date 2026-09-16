# Ticketing & Access Domain

## Domain Overview

Ticketing & Access owns the **commercial lifecycle of estate admission**: product catalog, cart management, dynamic pricing, payment processing, credential issuance, gate validation, and the AI Guest Concierge. It is the revenue engine of the estate.

**Key Invariant:** The Commerce Module is the sole authority for prices, inventory, and financial transactions. The AI Concierge has read-only access to catalog and availability — it can suggest but never commit.

---

## Module Decomposition

| Module | Responsibility | AI Involvement |
|---|---|---|
| **Catalog** | Product types, ticket types, descriptions | None (reference data) |
| **Cart** | Active shopping sessions (Redis-backed) | None |
| **Pricing Engine** | Base prices + dynamic adjustments within ±30% bands | Consumes demand forecast signal from Park Ops |
| **Commerce** | Order creation, payment delegation, refund processing | None (deterministic) |
| **Credential** | Ed25519 JWT signing, QR generation, revocation management | None (cryptographic) |
| **Gate Validator** | Online validation + offline Ed25519 fallback | None (deterministic) |
| **Capacity** | Redis atomic counters per date/slot, reservation tracking | None |
| **AI Concierge** | NL conversation → tool calls to Catalog/Capacity | GenAI (tool-calling LLM via Model Gateway) |
| **Next Best Action** | Post-purchase upsell recommendations | ML ranking + GenAI copy drafting |
| **Fraud Scoring** | Transaction risk evaluation | ML classification model |

---

## AI Guest Concierge — Detailed Design

### Interaction Flow

```mermaid
sequenceDiagram
    autonumber
    participant Visitor
    participant UI as Web / Mobile UI
    participant Concierge as AI Guest Concierge
    participant MG as Model Gateway
    participant LLM as LLM Provider
    participant Catalog as Catalog Module
    participant Capacity as Capacity Module
    participant Commerce as Commerce (Deterministic)
    participant Pay as Payment Provider

    Visitor->>UI: "Family of 4, next Saturday, best deal?"
    UI->>Concierge: Forward utterance + session context
    Concierge->>MG: Task: concierge.ticket.chat
    MG->>LLM: Prompt + tools (search_products, check_availability)

    LLM->>Catalog: tool_call: search_products(family, 4 pax)
    Catalog-->>LLM: Family Pass $119, Individual 4×$196

    LLM->>Capacity: tool_call: check_availability(2026-09-14)
    Capacity-->>LLM: Saturday: 80% sold, 10AM slot available

    LLM-->>Concierge: "Family Pass saves $77. Book 10AM?"
    Concierge-->>UI: Display response to visitor

    Visitor->>UI: "Yes, book it"
    UI->>Commerce: Create cart (Family Pass, 2026-09-14, 10AM)
    Commerce->>Commerce: Validate price against catalog (NOT LLM quote)
    Commerce->>Capacity: Reserve inventory (idempotent)
    Commerce-->>UI: Cart confirmed. Price: $119.00

    Note over UI,Commerce: Price shown is ALWAYS from Commerce, never from LLM

    Visitor->>UI: Proceed to payment
    UI->>Commerce: Initiate payment ($119.00)
    Commerce->>Pay: Charge (idempotent key: ord-xxx)
    Pay-->>Commerce: Payment confirmed
    Commerce->>Commerce: Issue signed QR credential
    Commerce-->>UI: Order confirmed + QR code
```

### Concierge Guardrails

| Guardrail | Mechanism | Fallback |
|---|---|---|
| Read-only tool access | Tool allowlist: `search_products`, `check_availability`. No write APIs. | Static catalog page |
| Per-session cost ceiling | $0.15 enforced by Model Gateway | Session terminated; redirect to catalog |
| Runaway session detection | > 10 turns | Conversation ended with summary + catalog link |
| Prompt injection defense | Input sanitization, system prompt integrity, no PII in context | Tool call fails gracefully |
| Price mismatch detection | Commerce compares LLM-quoted price against catalog at checkout | Correct price always displayed |

---

## Gate Entry — Online-First with Offline Fallback

```mermaid
sequenceDiagram
    autonumber
    participant Visitor as Visitor (QR on Phone)
    participant Scanner as Gate Scanner (Edge)
    participant Cloud as Ticket Validation API
    participant Bloom as Local Bloom Filter
    participant EntryLog as Local Entry Log (LiteDB)

    Visitor->>Scanner: Present QR code
    Scanner->>Scanner: Decode JWT from QR

    alt WiFi Available (Online Mode)
        Scanner->>Cloud: Validate ticket_id (real-time)
        Cloud-->>Scanner: VALID (not revoked, not used, capacity OK)
        Scanner->>Scanner: Open turnstile
        Scanner->>EntryLog: Record entry locally
        Scanner->>Cloud: Report VisitorEntered event
    else WiFi Unavailable (Offline Fallback)
        Scanner->>Scanner: Verify Ed25519 signature (local public key)
        Scanner->>Scanner: Check expiry and valid_date
        Scanner->>Bloom: Check ticket_id in revocation Bloom filter
        Bloom-->>Scanner: NOT FOUND (not cancelled)
        Scanner->>EntryLog: Check for prior entry today
        EntryLog-->>Scanner: NOT FOUND (first entry)
        Scanner->>Scanner: VALID — open turnstile
        Scanner->>EntryLog: Record entry (buffered for sync)
        Note over Scanner,Cloud: Events synced when WiFi returns
    end
```

---

## Dynamic Pricing Flow

```mermaid
sequenceDiagram
    autonumber
    participant Weather as Weather API
    participant Sales as Sales History
    participant Calendar as Calendar/Events
    participant Forecast as Demand Forecast (Park Ops)
    participant Advisor as Dynamic Pricing Advisor
    participant Rules as Pricing Rules Engine
    participant Catalog as Catalog Module

    Note over Forecast: Batch run: nightly + triggered on significant change

    Weather->>Forecast: Saturday forecast: sunny, 24°C
    Sales->>Forecast: Last 4 Saturdays avg: 4,200 sold
    Calendar->>Forecast: School holidays: YES

    Forecast->>Forecast: Predict: 5,800 visitors (high confidence)
    Forecast->>Advisor: Predicted demand: HIGH for Saturday

    Advisor->>Rules: Recommend: +15% for Saturday peak slots
    Rules->>Rules: Validate within guardrails (max +30%, min -20%)
    Rules->>Catalog: Update Saturday peak pricing: $56 (was $49)

    Note over Rules,Catalog: Pricing rules engine is deterministic
    Note over Rules,Catalog: AI forecast is input signal only
    Note over Rules,Catalog: Human-approved bands prevent runaway pricing
```

---

## Core Data Entities

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--|{ TICKET : contains
    ORDER ||--o| PAYMENT : settles
    TICKET ||--o{ ENTRY_EVENT : validates
    TICKET }o--|| TICKET_TYPE : categorizes
    TICKET ||--o{ TIMED_ENTRY : includes
    TICKET_TYPE }o--|| PRICING_RULE : governed_by
    ORDER ||--o{ OFFER_APPLIED : uses

    CUSTOMER {
        string customer_id PK
        string email
        string name
        string phone_hash
        string consent_status
        float return_propensity
        timestamp created_at
    }

    ORDER {
        string order_id PK
        string customer_id FK
        string status
        decimal total_amount_usd
        string channel
        timestamp created_at
    }

    TICKET {
        string ticket_id PK
        string order_id FK
        string ticket_type_id FK
        string access_level
        date valid_date
        string time_slot
        int party_size
        string credential_jti
        string status
    }

    TICKET_TYPE {
        string ticket_type_id PK
        string name
        boolean is_family_pass
        int max_party_size
    }

    PRICING_RULE {
        string rule_id PK
        string ticket_type_id FK
        decimal base_price_usd
        decimal min_price_usd
        decimal max_price_usd
        decimal current_adjustment_pct
    }

    PAYMENT {
        string payment_id PK
        string order_id FK
        string provider_ref
        decimal amount_usd
        string status
        string idempotency_key
    }
```

---

## Published CloudEvents & Inbound Commands

All events use CloudEvents 1.0 format with ULID `id` and PascalCase `.v1`:

| Published Event (`type`) | Consumers | Key Data Fields | Architectural Impact |
|---|---|---|---|
| `TicketPurchased.v1` | Park Ops, Growth | `orderId`, `ticketIds[]`, `partySize`, `visitDate`, `timeSlot`, `totalAmountUsd` | Park Ops updates presale inputs for demand forecast; Growth starts session. |
| `TicketCancelled.v1` | Park Ops, Growth | `orderId`, `ticketIds[]`, `reason`, `refundAmountUsd` | Forecast adjustments; revocation Bloom filter updated. |
| `TicketRescheduled.v1` | Park Ops, Growth | `orderId`, `originalDate`, `newDate`, `newTimeSlot` | Daily capacity reallocation. |
| `VisitorEntered.v1` | Park Ops, Growth, Living Collection | `ticketId`, `gateId`, `zoneId`, `scannedAt`, `partySize` | Authoritative in-park occupancy increment; starts companion session. |
| `VisitorExited.v1` | Park Ops | `ticketId`, `gateId`, `scannedAt` | Decrements live occupancy count. |
| `CapacityThresholdReached.v1` | Park Ops | `date`, `timeSlot`, `currentSold`, `capacityLimit`, `utilizationPct` | Correlates with crowd forecast and dynamic pricing advisor. |

> **No private timed-entry ledger.** Ticketing sells exhibit windows but does not allocate them. A pre-visit exhibit reservation is published as `VisitRequested.v1` with `origin: pre_visit` and is confirmed — or refused, with alternates — by Living Collection, the sole capacity authority. The exhibit window is only printed on the credential once `VisitConfirmed.v1` returns, so a slot sold at checkout and a slot claimed in-park draw down the same cap.

---

## Failure Scenarios & Mitigations

| Failure | Response | Pattern |
|---|---|---|
| Payment provider down | Hold `pending_payment`; retry with backoff; offer alternate provider | Retry + Circuit Breaker |
| LLM provider down | Gateway failover: secondary → local SLM → static catalog UI | 4-Tier Failover |
| WiFi outage at gates | Full offline mode: Ed25519 + Bloom filter + LiteDB | Edge Autonomy |
| Peak holiday traffic (3×) | Auto-scaling + queue-based checkout (virtual FIFO queue) | Load Leveling |
| Duplicate payment | Idempotency key on every payment request | Idempotency |
| Bloom filter stale | Worst case: one cancelled ticket admitted before 5-min sync | Acceptable Trade-off |

---

## Domain ADRs

- [ADR-003: Offline Cryptographic Ticket Validation](../adrs/ADR-003-offline-cryptographic-ticket-validation.md)
- [ADR-005: Living Collection as Sole Capacity Authority](../adrs/ADR-005-living-collection-as-sole-capacity-authority.md)
- [ADR-013: Modular Monolith per Bounded Context](../adrs/ADR-013-modular-monolith-per-bounded-context.md)
- [ADR-014: Dynamic Pricing Hard Guardrails](../adrs/ADR-014-dynamic-pricing-hard-guardrails.md)
