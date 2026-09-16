# Glossary — Ubiquitous Language

> Cross-context terms used consistently across all four bounded contexts. Terms are organized by domain first, then by cross-cutting concepts.

---

## Ticketing & Access

| Term | Definition |
|---|---|
| **Ticket** | A cryptographically signed credential (Ed25519 JWT) granting access to the estate for a specific date and optional time slot |
| **Family Pass** | A ticket type covering up to N party members under a single order at a discounted rate |
| **Credential** | The signed QR code (JWT) issued upon purchase; verified at gates via signature, not API call |
| **Commerce Module** | The deterministic subsystem that owns pricing, inventory reservation, cart, and payment. Sole authority for all financial operations |
| **Concierge** | The AI Guest Concierge — a GenAI agent that helps visitors explore and purchase tickets via natural language. Has read-only tool access to catalog and availability; cannot invoke payment or credential APIs |
| **Dynamic Pricing Advisor** | An AI model that recommends price adjustments based on demand forecasts, constrained by human-approved ±30% guardrails |
| **Bloom Filter** | A probabilistic data structure cached on gate scanners containing revoked ticket IDs. Synced every 5 minutes. False-positive rate: one wrongly-rejected ticket per ~10 million scans |
| **Spot Saver** | The estate's single guest-facing virtual queue, covering both mechanical rides and animal exhibits. Ride positions are held by Guest Growth and signalled to Park Operations; every exhibit reservation — pre-visit or in-park — originates as `VisitRequested.v1` and confirms as `VisitConfirmed.v1` against Living Collection's single capacity ledger |
| **Pre-Authorized Slab** | A non-overlapping range of ticket IDs assigned to a kiosk for offline sales |

## Living Collection

| Term | Definition |
|---|---|
| **Enclosure** | A physical habitat for one or more animal species, identified as `ENC-{TYPE}-{NN}` (e.g., `ENC-AMAZON-17`) |
| **Clinical Projection** | The internal, vet-only view of the Digital Twin containing raw telemetry, feeding curves, drug dosages, necropsy data |
| **Public Projection** | The guest-facing view of the Digital Twin exposing only `availability`, `population_band`, `viewing_hint`, `story_card` |
| **School Aggregate** | The domain model for untaggable species (e.g., jumping piranhas). Contains estimated headcount n̂ with confidence interval [ci_low, ci_high], biomass $B = \hat{N} \cdot \bar{w}$, dispersion index $D$, and velocity coherence $C_v$ |
| **Keeper Disposition** | A keeper's judgment on an AI inference: `accept`, `reject`, or `correct` (with manual override value). Used as production ground truth |
| **Winkler Score** | A statistical measure of inter-rater reliability between AI model predictions and keeper dispositions, computed per enclosure per week |
| **Edge CV** | Computer vision inference running on edge hardware (Jetson Orin Nano). Processes 5-frame camera bursts into school aggregate counts |
| **Pixel Sovereignty** | The principle that camera frames stay on estate hardware by default. Media leaves the estate only on explicit keeper pull request, after automated facial de-identification |
| **Welfare-Safe Filter** | A neural gate that evaluates visitor-submitted photos for animal welfare violations (flash, barrier breach, foreign objects) |

## Park Operations

| Term | Definition |
|---|---|
| **Zone** | A logical area of the estate (55 total) containing multiple Places, classified into one of 5 `zone_type` biomes. Zones have adjacency relationships maintained by hand |
| **Place** | A specific location within a Zone: a ride, queue lane, path segment, or exhibit |
| **Popularity Score** | A weighted composite of four signals (zone occupancy, ride cycle throughput, queue wait time, virtual queue depth) per Place |
| **Demand Forecast** | The estate's single multi-horizon gradient-boosted ML model, owned by Park Operations, predicting tactical (15m/2h) and strategic (14d) occupancy curves from shared estate signals |
| **Staffing Recommendation** | A system-generated suggestion to move N keepers from Zone X to Zone Y, with reason and numbers. Always requires human Approve/Reject/Modify |
| **Silence Detector** | An operational diagnostic algorithm distinguishing genuine low popularity from sensor failure or unannounced ride mechanical downtime |
| **CapEx Flag** | A stateless indicator surfaced in monthly investment reports when a Place shows sustained low popularity or sustained high popularity with overflow |
| **Shift Briefing** | A GenAI-generated narrative summarizing current zone status, recommendations, and forecasts. Every number is faithfulness-verified against source data |
| **Faithfulness Check** | A deterministic verification that every number, zone name, and entity referenced in GenAI-generated text is traceable to the structured input data |
| **Signal** | A time-series data point from an edge sensor: people-count, ride-cycle, queue-occupancy, or environmental reading |
| **Baseline Estimate** | A cloud-computed historical average used as a substitute when a sensor is Silent or Anomalous |

## Guest Growth

| Term | Definition |
|---|---|
| **Companion App** | An offline-first PWA providing the guest experience: ticket wallet, virtual queue, recommendations, and estate map |
| **Spot Saver Facade** | The client-facing reservation component in Guest Growth. Submits `VisitRequested.v1` to Living Collection (capacity authority) and renders passes |
| **Honesty Invariants** | Absolute UX constraints: the companion app can never display a wait time lower than Park Operations sensor reality, nor recommend a closed exhibit |
| **Next Best Action (NBA)** | A recommendation engine that evaluates guest context (time remaining, party composition, current zone, crowd state) to suggest the optimal next attraction |
| **Nudge** | A personalized push notification or in-app message designed to increase engagement, return visits, or ancillary spend. Subject to consent and congestion checks |
| **Return Propensity** | A score [0, 1] predicting the likelihood of a visitor returning within 90 days, computed by a churn prediction model |

## Cross-Cutting

| Term | Definition |
|---|---|
| **Bounded Context** | A DDD concept: a clear boundary within which a domain model has a consistent, well-defined meaning. This platform has four: Ticketing & Access, Living Collection, Park Operations, Guest Growth |
| **Model Gateway** | The shared infrastructure component through which all four contexts invoke AI capabilities. Routes task IDs to providers, enforces cost ceilings, manages failover chain |
| **Task ID** | A logical, namespaced name for an AI capability (e.g., `tk.concierge.ticket.chat`, `lc.census.school.density`, `po.shift.briefing`). Domains call task IDs; the gateway routes to models |
| **4-Tier Failover** | Primary Cloud LLM → Secondary Cloud LLM → Local SLM (Mistral-7B on estate cluster) → Static template/raw data |
| **CloudEvents 1.0 (ULID)** | Standard CNCF event envelope. `id` is a ULID used as idempotency key; `type` follows PascalCase `.v1` convention |
| **Cluster Gateway Pair** | Industrial compute nodes (6 pairs estate-wide) running Mosquitto/EMQX, serving all 55 zones with 72h NVMe store-and-forward buffers; one pair carries both animal telemetry and ride/queue sensing for its perimeter |
| **Three-VLAN Segmentation** | VLAN 10/20 (Operations & IoT sensors, mTLS), VLAN 30 (Admin/Back-office), VLAN 50 (Guest DMZ Wi-Fi & Beacons). Inter-VLAN routing is dropped |
| **Golden Set** | A curated dataset of annotated inputs and expected outputs used to evaluate AI model quality before deployment |
| **Transactional Outbox** | A pattern where domain events are written to the same database transaction as business state, guaranteeing at-least-once publishing to Kafka |
