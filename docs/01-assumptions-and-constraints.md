# Assumptions & Constraints

## Physical Estate Assumptions

| # | Assumption | Impact on Design |
|---|---|---|
| A1 | **40 heritage rides** are fixed-capacity mechanical attractions with known throughput rates | Ride-cycle events are discrete (per dispatch), not continuous |
| A2 | **200+ animals across 55 enclosures** and grounds map to **5 logical `zone_type` classifications** | Standardized sensing and compute profiles allocated per zone type: |
| | • `terrestrial` (serpent house, salamanders) | Air temp, humidity, lux, RFID, load-cell feeders, thermal cams |
| | • `aquatic` (Amazon House, jumping piranhas) | Dissolved oxygen (DO), pH, water level, hydrophones, submersible cams |
| | • `nocturnal` (nocturnal house, carnivorous flora) | Low-light/thermal cameras, door interlocks, red lux spectrum |
| | • `recreational` (Gnome Coaster, carousel, midways) | Ride-cycle PLCs, queue cameras, optical people counters |
| | • `mixed` (rose garden paths, retail plazas) | People counters, BLE beacons, interactive guest kiosks |
| A3 | Physical edge hardware is sized per **zone cluster** rather than 55 redundant servers | **6 shared cluster gateway pairs** (Mosquitto/EMQX + 72h NVMe) serve all 55 zones across Living Collection & Park Ops |
| A4 | **Entry gates** are finite chokepoints (4–6 initially, scaling to 10–15) | Gate scanners need independent offline Ed25519 validation (<50ms P99) |
| A5 | Grounds include outdoor paths, indoor exhibit halls, and mixed areas | BLE beacon placement and density varies by area type on Guest VLAN 50 |
| A6 | Estate boundary is secured — entry/exit requires passing a gate | Turnstile count is authoritative for in-park occupancy |

## Network & Infrastructure Constraints

| # | Constraint | Source | Architecture Response |
|---|---|---|---|
| C1 | **WiFi is patchy** across the estate | Problem statement | Edge-first MQTT 5 mesh, 72h NVMe buffer, store-and-forward |
| C2 | Cloud services available but **no guaranteed uplink** | Problem statement | All critical paths have offline fallback |
| C3 | **Budget for MQTT-capable hardware** exists | Problem statement | Full edge sensor mesh is in scope |
| C4 | Estate is a **historic site** — infrastructure changes must be non-destructive | Implied | Wireless sensors, BLE beacons, no trenching for fiber |
| C5 | **No 5G coverage** assumed on estate grounds | Conservative assumption | 4G SIM failover for edge gateways |

## Visitor & Scale Constraints

| # | Constraint | Source | Architecture Response |
|---|---|---|---|
| S1 | Current load: **5,000 visitors/day** | Problem statement | Baseline capacity planning |
| S2 | Growth target: **15,000 visitors/day** within 3 years | Problem statement | 3× capacity headroom in all designs |
| S3 | Visitors include families with children, elderly, and international tourists | Implied | Multi-language AI Concierge, family pass logic, accessibility in companion app |
| S4 | Some visitors will not have smartphones | Implied | Physical kiosk parity, no phone-only features |
| S5 | Peak-to-average ratio approximately 3:1 (holidays, weekends) | Industry standard | Auto-scaling and queue-based load leveling |

## AI & Model Constraints

| # | Constraint | Source | Architecture Response |
|---|---|---|---|
| M1 | AI model providers may **change pricing** at any time | Problem statement | Task-based Model Gateway with cost ceilings |
| M2 | AI model providers may **shut down** entirely | Problem statement | 4-tier failover chain |
| M3 | GenAI outputs are **non-deterministic** | Fundamental | Commerce is deterministic, AI is assistive only |
| M4 | **No current labeled dataset** for piranha counting | Realistic startup | Transfer learning from open fisheries datasets + keeper disposition feedback loop |
| M5 | **Per-session AI cost ceiling: $0.15** | Architecture decision | Model Gateway enforces hard cutoff |

## Regulatory & Legal Assumptions

| # | Assumption | Impact on Design |
|---|---|---|
| R1 | GDPR / UK GDPR compliance required | PII never in domain events; phone/email hashed at rest; AI models receive anonymized profiles |
| R2 | PCI DSS for payment processing | Delegated to Stripe/Adyen; no raw card storage |
| R3 | **CITES / ZAA animal welfare regulations** apply | Full audit trail for animal health events; keeper accountability |
| R4 | No live-streaming of enclosure cameras to public | Media stays on estate (pixel sovereignty); faces blurred at edge |
| R5 | Estate must display privacy notices for camera areas | Signage obligation, outside architecture scope |

## Explicit Non-Constraints

These are things we explicitly choose **not** to constrain:

- **Cloud provider**: Architecture is cloud-agnostic (containerized, Kafka-based backbone)
- **LLM provider**: Model Gateway abstracts provider choice
- **Programming language**: Not specified — each domain team chooses (modular monolith allows polyglot at the domain boundary)
- **Mobile framework**: Companion app is a PWA (no native app store dependency)
