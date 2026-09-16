# Non-Goals — What This Architecture Deliberately Excludes

> Architectural discipline is as much about what you say no to as what you build. Each non-goal is accompanied by the reasoning for its exclusion.

---

## Out of Scope

| # | Non-Goal | Why Excluded |
|---|---|---|
| NG-1 | **Autonomous AI decision-making** | AI assists every decision but never acts alone. Commerce is deterministic. Safety is firmware-level. Staff deployment is human-approved. The architecture is explicitly designed so that removing all AI degrades quality but never breaks operations. |
| NG-2 | **Live video streaming to cloud** | 55 enclosures × continuous video = unjustifiable bandwidth and cloud cost. Pixel sovereignty means frames stay on estate hardware. Media leaves only on explicit keeper pull request, after automated facial de-identification. |
| NG-3 | **Individual animal tagging for schooling species** | You cannot tag a jumping piranha. The architecture models schooling fish as a School Aggregate with estimated n̂ and confidence interval. Pretending otherwise would produce fabricated individual IDs. |
| NG-4 | **Microservices per capability** | At 5K–15K visitors/day, modular monolith per bounded context is the right granularity. Microservices would introduce distributed transaction complexity disproportionate to the scale. Each module can be extracted later if needed. |
| NG-5 | **Native mobile apps** | The companion app is a PWA (Progressive Web App). No App Store/Play Store dependency. Offline-first via service workers and LiteDB. International visitors don't need to install anything. |
| NG-6 | **Social media scraping** | Visitor media is ingested only via first-party QR uploads or official platform mention APIs. Zero social media scraping. Protects guest privacy and avoids legal liability. |
| NG-7 | **Automatic threshold tuning** | Staffing recommendation thresholds, pricing guardrails, and alert sensitivity are all human-configured. Auto-tuning would reintroduce the opacity that deterministic rules layers were chosen to avoid. |
| NG-8 | **Real-time staff dispatch** | The system recommends staff moves but does not dispatch. The duty manager radios the move via existing ops process. The system is not the manager. |
| NG-9 | **Automated refunds** | Refunds above configurable thresholds require dual human approval. The system surfaces the recommendation; a human authorizes the financial action. |
| NG-10 | **Replacing the estate WiFi** | The architecture treats patchy WiFi as a permanent constraint, not a problem to solve. Every design assumes WiFi will fail. This is more resilient than depending on a WiFi upgrade. |
| NG-11 | **Multi-estate federation** | This architecture serves a single estate. Multi-site replication, cross-estate data sharing, and federated identity are out of scope. |
| NG-12 | **Guest facial recognition** | No facial recognition of guests, anywhere, ever. Camera-based counting uses silhouette detection. Any accidental face capture is blurred at the edge before storage. |

---

## Globally Forbidden Inbound Commands

To guarantee strict bounded context integrity and physical safety, the platform enforces hard cross-domain boundary rules. The following inbound commands are **architecturally and cryptographically forbidden** estate-wide:

| Forbidden Action | Target Subsystem | Why Architecturally Forbidden | Enforcing Mechanism |
|---|---|---|---|
| **External Price Override** | Ticketing & Access | No external agent or recommendation engine can set, alter, or negotiate admission ticket prices. | Dynamic pricing is strictly clamped inside Ticketing domain; AI Concierge is read-only. |
| **Direct Payment / Refund Issuance** | Ticketing & Access | No external system or AI model can directly trigger card debits or unreviewed cash refunds. | Commerce module is sole authority; refunds > €50 require dual human approval. |
| **Physical Gate / Solenoid Unlock** | Ticketing & Park Ops | Cloud systems and AI agents cannot remotely actuate physical turnstiles or gate unlock relays. | Turnstiles unlock exclusively via local cryptographic scan verification (VLAN 20). |
| **Feeder Actuation or Dosing Override** | Living Collection | No software algorithm or remote API can actuate animal feeding mechanisms or chemical dosing. | Physical feeding is manually initiated by keepers; telemetry is read-only. |
| **Water Chemistry Setpoint Modification** | Living Collection | No cloud service can alter aquarium heater, pump, or dissolved oxygen life-support thresholds. | SCADA controllers run autonomous PID loops on air-gapped VLAN 10. |
| **Biological Quarantine Override** | Living Collection | Guest Growth or Park Operations cannot reopen a closed exhibit or lift a veterinary quarantine. | Living Collection is sole capacity authority; quarantine clamps are immutable by external contexts. |
| **Automated Staff Dispatch** | Park Operations | Systems cannot automatically order staff reassignments or alter labor contracts. | Park Ops surfaces recommendations; duty managers approve and radio staff manually. |

---

## Deferred to Phase 2+

| # | Deferred Item | Phase | Why Deferred |
|---|---|---|---|
| D-1 | **Annual pass management** | Phase 2 | Requires loyalty program integration and visit-frequency tracking not in initial scope |
| D-2 | **Multi-language AI Concierge** | Phase 2 | Initial launch in English; multilingual prompts and response validation add complexity |
| D-3 | **Merchandise & F&B integration** | Phase 3 | Ticketing domain focuses on admissions first; ancillary commerce is a separate integration |
| D-4 | **Predictive maintenance for rides** | Phase 3 | Requires vibration sensor data collection over 6+ months for baseline |
| D-5 | **AR/VR experiences** | Future | Interesting but not architecturally significant for the current mandate |
