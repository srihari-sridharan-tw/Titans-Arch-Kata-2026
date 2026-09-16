# Architectural Characteristics

## Quality Attribute Prioritization

The Von Digitalis Estates platform optimizes for different quality attributes across different contexts. The table below shows the **relative priority** of each attribute per bounded context:

| Architectural Characteristic | Ticketing & Access | Living Collection | Park Operations | Guest Growth | Estate Overall |
|---|---|---|---|---|---|
| **Safety & Biological Integrity** | ⬛⬛ Low | ⬛⬛⬛⬛⬛ Critical | ⬛⬛⬛⬛ High | ⬛ Minimal | **#1 Priority** |
| **Availability & Offline Resilience** | ⬛⬛⬛⬛⬛ Critical | ⬛⬛⬛⬛⬛ Critical | ⬛⬛⬛⬛ High | ⬛⬛⬛ Medium | **#2 Priority** |
| **Data Integrity & Financial Consistency** | ⬛⬛⬛⬛⬛ Critical | ⬛⬛⬛⬛ High | ⬛⬛⬛ Medium | ⬛⬛ Low | **#3 Priority** |
| **Evolvability & Model Agnosticism** | ⬛⬛⬛ Medium | ⬛⬛⬛ Medium | ⬛⬛⬛ Medium | ⬛⬛⬛⬛⬛ Critical | **#4 Priority** |
| **Observability & Auditable Lineage** | ⬛⬛⬛⬛ High | ⬛⬛⬛⬛⬛ Critical | ⬛⬛⬛⬛ High | ⬛⬛⬛ Medium | **#5 Priority** |
| **Performance & Latency** | ⬛⬛⬛⬛ High | ⬛⬛⬛ Medium | ⬛⬛⬛ Medium | ⬛⬛⬛⬛ High | Operational Standard |
| **Cost Efficiency** | ⬛⬛⬛ Medium | ⬛⬛ Low | ⬛⬛⬛ Medium | ⬛⬛⬛⬛ High | Fused Constraints |

---

## Authoritative Driving Characteristics (Ranked Order)

### 1. Safety & Biological Integrity (Estate Rank #1)
**Rationale:** 200+ exotic and venomous species in proximity to up to 15,000 daily guests. A containment breach or water quality failure is a life-safety catastrophe. Animal welfare must never be compromised for commercial throughput.
**Architecture Response:**
- **Zero Inbound AI Actuation**: AI models output advisories and anomaly scores; deterministic rule engines enforce safety tripwires.
- **Physical Isolation**: Life-safety solenoids, magnetic habitat locks, and life-support PLCs are isolated behind physical optical data diodes and dry contacts (no IP-routable ingress).
- **Sole Capacity Authority**: Living Collection alone controls viewing gallery slots via `VisitRequested.v1` $\to$ `VisitConfirmed.v1`; commerce and guest-experience systems book against it, never around it.
- **Clinical/Public Twin Split**: Vets receive full diagnostic telemetry; public interfaces expose strictly sanitized welfare summaries.

### 2. Availability & Offline Resilience (Estate Rank #2)
**Rationale:** A guest arriving in heavy rain with valid tickets must never be stranded due to a WAN fiber cut. Enclosure monitoring must survive a 72-hour cloud disconnect.
**Architecture Response:**
- **Offline Turnstile Verification**: Ed25519 asymmetric cryptography validates tickets locally in <200ms using cached public keys.
- **72-Hour NVMe Buffer**: 6 cluster gateway pairs buffer all zone telemetry locally on high-endurance NVMe drives.
- **VRRP High Availability**: Dual active-passive Mosquitto/EMQX brokers per cluster pair fail over automatically in <3s.
- **Multi-Model Gateway Failover**: 4-tier provider cascade ensures conversational Concierge continuity during primary API outages.

### 3. Data Integrity & Financial Consistency (Estate Rank #3)
**Rationale:** Financial transactions, capacity quotas, and veterinary medical logs require absolute consistency. Double-selling slots or charging erroneous prices destroys estate credibility.
**Architecture Response:**
- **Single Price Authority**: Commerce engine governs final transactional checkout; dynamic pricing models output advisory recommendations only.
- **ULID Idempotency**: All CloudEvents 1.0 payloads carry lexicographically sortable ULID keys for exactly-once processing deduplication.
- **Transactional Outbox**: Local database state changes and Kafka event publications commit atomically to prevent ghost writes.
- **Atomic Capacity Counters**: Redis CAS (compare-and-swap) prevents race conditions during surge bookings.

### 4. Evolvability & Model Agnosticism (Estate Rank #4)
**Rationale:** The foundation model landscape changes monthly. Hardcoding specific proprietary model APIs directly into business logic leads to rapid obsolescence and vendor lock-in.
**Architecture Response:**
- **Task-Based Abstraction**: Domain services call namespaced Task IDs (`lc.*`, `tk.*`, `po.*`, `gr.*`), completely insulated from underlying LLM/SLM provider APIs.
- **CloudEvents 1.0 Schema Versioning**: PascalCase `.vN` contracts permit seamless schema evolution across domain boundaries.
- **Modular Monolith Foundation**: In-process domain separation allows selective microservice extraction without architectural refactoring.

### 5. Observability & Auditable Lineage (Estate Rank #5)
**Rationale:** Automated intelligence across 55 physical zones requires complete operational transparency, auditable decision lineage, and rapid failure detection.
**Architecture Response:**
- **Silence Detector**: Automated watchdog flags missing heartbeats from any zone sensor or gateway within 3 missed intervals.
- **Dead Letter Exchanges (DLX)**: Malformed or unprocessable events divert to isolated DLX topics with full diagnostic context.
- **OpenTelemetry & Tracing**: Distributed trace IDs propagate across edge MQTT, Kafka, and Model Gateway calls.

---

## Performance SLOs

| Operation | SLO | Architecture Mechanism |
|---|---|---|
| Gate scan (online) | < 500ms | Cloud API with Redis cache |
| Gate scan (offline) | < 200ms | Local Ed25519 signature verification |
| Ticket purchase (end-to-end) | < 3s | Optimistic capacity reservation |
| AI Concierge response | < 3s per turn | Model Gateway streaming |
| Edge CV piranha count | < 200ms per burst | Jetson Orin Nano local inference |
| Demand forecast cycle | < 15 min | Batch computation, cached results |
| Companion app page load | < 1s | Service worker caching, CDN |
| Push notification delivery | < 30s | Best-effort, non-critical |

---

## Scalability Profile

| Workload / Traffic Vector | Baseline (5K guests/day) | Peak Surge (15K guests/day) | Scaling Mechanism |
|---|---|---|---|
| **Gate Scans** | ~200 req/min | ~600 req/min | Auto-scaling API instances + edge Ed25519 fallback |
| **Ticket Purchases** | ~30 txn/min | ~100 txn/min | Queue-based leveling + optimistic capacity reservation |
| **AI Concierge Sessions** | ~20 /min | ~60 /min | Horizontal Model Gateway scale + multi-provider pool |
| **Edge Telemetry** | ~500 msg/min | ~1,500 msg/min | Kafka partition distribution + NVMe edge buffers |
| **Companion App Traffic** | ~100 req/min | ~300 req/min | Edge CDN caching + app service worker |

The architecture scales horizontally at every tier:
- **Edge:** Add more zones with their own gateways
- **Cloud:** Auto-scale modular monolith instances
- **AI:** Model Gateway scales horizontally; LLM provider handles compute
- **Data:** Kafka partitioning; TimescaleDB hypertables; Redis cluster mode
