# Living Collection Domain

## Domain Overview

Living Collection owns the **welfare, monitoring, and public representation** of 200+ exotic animals across 55 enclosures. This is the highest-safety domain: a missed containment alert on a venomous snake or a piranha escape is a life-threatening emergency.

**Key Invariants:**
1. LLM is **never** in the safety path. Safety is deterministic; health detection is specialized ML; LLMs draft keeper notes and guest stories only.
2. Living Collection is the **sole capacity authority** for enclosure timed-entry windows.
3. The **Clinical Projection** (vet-only) and **Public Projection** (guest-facing) are strictly separated at the aggregate boundary.

---

## Module Decomposition

| Module | Responsibility | AI Involvement |
|---|---|---|
| **Telemetry Ingest** | MQTT → time-series storage. Deduplication, bounds-checking. | None |
| **Inventory** | Species catalog, enclosure registry, individual/school aggregates | None |
| **Husbandry** | Feeding schedules, medication, care logs, round records | None |
| **Clinical Twin** | Full veterinary data: telemetry curves, drug dosages, necropsy data | Anomaly detection (ML) |
| **Public Twin** | Sanitized projection: `availability`, `population_band`, `viewing_hint`, `story_card` | None (data projection) |
| **Inference** | Edge CV (piranha count), appetite anomaly, behavioral anomaly | Edge CV + ML models |
| **Alert Engine** | Deterministic policy: AI probabilities → operational rules → alert actions | None (deterministic rules) |
| **Keeper Copilot** | Draft clinical notes from SOP + 14-day twin data (RAG) | GenAI via Model Gateway |
| **Story Narrator** | Generate guest-facing enclosure stories from public twin data | GenAI via Model Gateway |
| **Feedback Store** | Immutable keeper disposition log (accept/reject/correct) | Used for retraining |

---

## Edge Computer Vision — Piranha Census

### Why School Aggregate?

Jumping piranhas (*Pygocentrus nattereri*) cannot be individually tagged. Dense schooling, continuous 3D locomotion, and water turbidity make individual identification biologically impossible. The domain models them honestly as a **School Aggregate**:

1. **Population Estimate Range**:
   - Rather than pretending to track each individual fish, the system estimates the total school count with a statistical confidence range (e.g., estimated 450 fish, within a 435–465 confidence band).

2. **Total Biomass**:
   ```text
   Total School Biomass = Estimated Fish Count × Rolling Average Weight
   ```
   *(where average fish weight is continuously updated from feeder load-cell telemetry and optical sizing)*.

3. **School Dispersion Index (Spacing & Clustering)**:
   Measures whether fish are clustered tightly or spread out across the aquarium:
   - **Clump / Feeding Frenzy (Score > 1.4)**: Fish are packed tightly in one area (normal during scheduled feedings).
   - **Lethargy / Hypoxia Warning (Score < 0.7)**: Fish are abnormally scattered and motionless (early indicator of low oxygen or water stratification).

4. **Velocity Coherence (Synchronous Swim Direction)**:
   Measures whether the school is swimming together in unison (scored from 0.0 to 1.0):
   - **Normal Schooling (Score < 0.50)**: Fish circulate independently in varied directions.
   - **Escape / Flight Reflex (Score > 0.85)**: Entire school darts in an identical direction simultaneously (indicates acute stress, pump vibration, or tank disturbance).

- Downstream systems consume **statistical population bands**, not fabricated individual IDs.
- If the Dispersion Index drops below 0.7 alongside a dissolved oxygen drop below 4.5 mg/L, an early-warning telemetry event fires before clinical distress occurs.

### Census Pipeline

```mermaid
stateDiagram-v2
    [*] --> IngestSnapshot: 5-Frame Camera Burst
    IngestSnapshot --> ConfidenceFilter: Calculate Count & Confidence Range
    
    ConfidenceFilter --> HighConfidence: Quality ≥ 0.85
    ConfidenceFilter --> LowConfidence: Turbid Water / Algae / Occlusion
    
    LowConfidence --> RequestSecondaryCamera: Cross-Correlate Angle
    RequestSecondaryCamera --> HighConfidence: Combined Fusion Meets Threshold
    RequestSecondaryCamera --> ManualInspectionQueued: Image Degradation Persists
    
    HighConfidence --> IntervalComparison: Compare Range vs. Day-1 Baseline
    
    IntervalComparison --> CensusStable: Confidence Ranges Overlap
    IntervalComparison --> DisjointDropDetected: Significant Population Drop
    
    DisjointDropDetected --> DoubleBlindKeeperCount: Shed Tablet Verification
    ManualInspectionQueued --> DoubleBlindKeeperCount: Physical Keeper Inspection
    
    DoubleBlindKeeperCount --> CensusStable: Keeper Confirms Normal Population
    DoubleBlindKeeperCount --> ContainmentInvestigation: Keeper Confirms Lost Fish
    
    ContainmentInvestigation --> CensusStable: Transfer / Natural Death Logged
    ContainmentInvestigation --> EscapedFishEmergency: Physical Escape Identified
```

### Edge Inference Details

| Parameter | Value | Rationale |
|---|---|---|
| Model | YOLOv8-nano (custom-trained) | Smallest model that fits Jetson Orin Nano |
| Input | 5-frame burst at 2 FPS | Captures jumping motion across frames |
| Output | Bounding boxes → estimated count & confidence range | Statistical aggregate, not individual IDs |
| Latency | < 200ms per burst | Fast enough for continuous monitoring |
| Training data | Open fisheries datasets + synthetic augmentation + keeper-labeled estate data | Transfer learning bootstrapping |
| Retraining trigger | Winkler score drops below 0.80 for 7 consecutive days | Production drift detection |

---

## Clinical vs. Public Twin

```mermaid
graph TB
    subgraph Clinical["Clinical Projection (Vet-Only)"]
        CT1["Raw Telemetry Curves"]
        CT2["Drug Dosages & Prescriptions"]
        CT3["Necropsy Records"]
        CT4["Feeding Tare Deltas"]
        CT5["Evidence Video Clips"]
        CT6["AI Anomaly Scores"]
    end

    subgraph Public["Public Projection (Guest-Facing)"]
        PT1["availability: open | closed | limited"]
        PT2["population_band: typical | growing | declining"]
        PT3["viewing_hint: most_active_morning"]
        PT4["story_card: GenAI-generated narrative"]
        PT5["capacity_cap: max 30 guests"]
    end

    Clinical -->|"Sanitized Projection<br/>(no clinical detail leaks)"| Public

    Ticketing["Ticketing"]
    Growth["Guest Growth"]
    Public -->|"ExhibitAvailabilityChanged"| Ticketing
    Public -->|"ExhibitAvailabilityChanged"| Growth

    classDef clin fill:#450a0a,stroke:#ef4444,stroke-width:2px,color:#f8fafc
    classDef pub fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#f8fafc
    classDef ext fill:#1e293b,stroke:#94a3b8,stroke-width:1px,color:#94a3b8

    class CT1,CT2,CT3,CT4,CT5,CT6 clin
    class PT1,PT2,PT3,PT4,PT5 pub
    class Ticketing,Growth ext
```

---

## Keeper Copilot — GenAI Clinical Note Drafting

```mermaid
sequenceDiagram
    autonumber
    participant Keeper as Keeper (Mobile)
    participant Copilot as Keeper Copilot
    participant MGW as Model Gateway
    participant RAG as RAG Context
    participant LLM as LLM Provider
    participant Twin as Clinical Twin

    Keeper->>Copilot: "Draft round note for ENC-AMAZON-17"
    Copilot->>Twin: Fetch 14-day telemetry summary
    Twin-->>Copilot: Feeding curve, water params, census history
    Copilot->>RAG: Retrieve relevant SOPs for piranha husbandry
    RAG-->>Copilot: SOP sections + care guidelines
    Copilot->>MGW: Task: keeper.copilot.draft
    MGW->>LLM: Prompt + twin data + SOPs
    LLM-->>MGW: Draft clinical note (markdown)
    MGW-->>Copilot: Draft note
    Copilot-->>Keeper: Display draft for review

    alt Keeper Accepts
        Keeper->>Copilot: Accept (one-tap)
        Copilot->>Twin: Save note as keeper-approved
    else Keeper Edits
        Keeper->>Copilot: Edit and save
        Copilot->>Twin: Save edited note + edit distance metric
    else Keeper Rejects
        Keeper->>Copilot: Reject (one-tap flag)
        Copilot->>Twin: Log rejection as training signal
    end
```

### Copilot Guardrails

- **No actuation**: Copilot cannot trigger feeding, medication, or containment actions
- **No diagnosis**: Copilot drafts notes; it does not diagnose. Veterinary judgment is sacrosanct
- **Clinical data only**: Copilot accesses the Clinical Twin, never the Public Twin in reverse
- **Faithfulness**: Every measurement in the draft must trace to the 14-day twin data

---

## Containment & Life-Safety Architecture

```mermaid
flowchart TD
    subgraph Loop["Local Containment Perimeter (Air-Gapped Life-Safety Loop)"]
        Sensor["Lid Contact Sensor<br/>(Normally-Closed Dry Contact)"] --> MCU["Enclosure Safety Microcontroller<br/>(Deterministic Firmware Logic)"]
        
        MCU -->|"Direct Dry Contact (<10ms)"| Solenoid["Physical Solenoid Interlock<br/>(Immediate Barrier Lock)"]
        MCU -->|"Relay Output (<20ms)"| Strobe["Local Strobe Siren<br/>(Audible & Visual Alert)"]
        MCU -->|"Hardwired Serial (<30ms)"| ShedTerminal["Shed Terminal & Tablet<br/>(Keeper Warning Display)"]
        MCU -->|"Local Ethernet (<50ms)"| LocalMQTT["Zone MQTT Broker<br/>(Buffered CloudEvent Dispatch)"]
    end
```

| Operational Characteristic | Specification | Architectural Safety Invariant |
|---|---|---|
| **Total Response Latency** | **< 50 ms** (firmware-level execution) | Immediate mechanical solenoid engagement before physical barrier breach |
| **Power Resilience** | Dedicated Zone UPS per enclosure | Continuous operation through complete estate electrical power outages |
| **Cloud Dependency** | **Zero** | 100% autonomous local execution; completely immune to WAN outages |
| **AI Dependency** | **Zero** | Pure deterministic relay & firmware logic; zero probabilistic models |
| **Wi-Fi Dependency** | **Zero** | Hardwired copper loop; immune to RF interference or packet loss |

---

## Visitor Media (UGC) — Ethical Side Channel

Visitors take photos of exhibits. This is an **untrusted, optional side channel** — if severed, core monitoring is completely unaffected.

| Control | Design |
|---|---|
| **Ingestion** | First-party exhibit QR upload + official platform mentions API only. Zero social media scraping |
| **Welfare gate** | Neural filter checks for flash, barrier breach, foreign objects. Violators silently dropped |
| **Privacy** | Automated guest face blur before any processing |
| **Rewards** | Non-monetary badges only ("Honorary Field Scout"). No cash bounties to prevent animal agitation |
| **Clinical impact** | Zero. Visitor media cannot write clinical state or trigger safety actions |

---

## Published CloudEvents
All events follow CloudEvents 1.0 with ULID `id` and PascalCase `.v1`:

| Published Event (`type`) | Consumers | Key Data Fields | Architectural Impact |
|---|---|---|---|
| `ExhibitAvailabilityChanged.v1` | Ticketing, Park Ops, Growth | `enclosureId`, `availability`, `capacityCap`, `reasonClass`, `estimatedReopening` | Ticketing halts sales; Spot Saver re-offers; Growth suppresses promotion. |
| `EnclosureUnsafe.v1` | Ticketing, Park Ops, Security | `enclosureId`, `code` (`lid_breach`/`barrier_down`), `occurredAt` | Park-wide safety alert; perimeter isolation; turnstiles seal zone. |
| `CensusUpdated.v1` | Park Ops, Keepers | `schoolId`, `nHat`, `ciLow`, `ciHigh`, `method` | PO correlates population density with visitor dwell times. |
| `FeedingCompleted.v1` | **Park Ops**, Keepers | `stationId`, `offeredG`, `consumedG`, `durationS` | PO uses for keeper labor utilization & staffing roster. |
| `StressIndexUpdated.v1` | **Park Ops** | `enclosureId`, `stressIndex` (0.0–1.0), `window` | PO throttles adjacent queue flow and moves staff. |
| `WelfareStoryReady.v1` | Guest Growth | `enclosureId`, `taxon`, `storyMarkdown`, `viewingTip` | Growth composes into itinerary cards without re-generation. |
| `BestViewingWindowPredicted.v1` | Guest Growth | `enclosureId`, `windowStart`, `windowEnd`, `confidence` | Growth itinerary planning; suggests which Spot Saver windows to offer first. |
| `VisitConfirmed.v1` | Guest Growth, Ticketing, Park Ops | `requestId`, `enclosureId`, `windowStart`, `windowEnd`, `partySize` | The authoritative slot grant against the welfare-clamped cap. No other context may issue one. |
| `VisitDenied.v1` | Guest Growth, Ticketing | `requestId`, `enclosureId`, `reasonClass` (`welfare`/`capacity`/`safety`), `alternativeWindows[]` | Refusal with alternates, so a booking client re-offers instead of failing the guest. |
| `SightingHonourable.v1` | Guest Growth | `contributorId`, `enclosureId`, `reasonCode` | Growth awards non-monetary digital badge. |

---

## Failure & Degraded-Mode Matrix

| Failure Mode | Edge Reaction | Cloud Reaction | Downstream Impact |
|---|---|---|---|
| Complete estate WAN outage | Edge buffers all telemetry (72h). Local sirens operational. | Mark gateways as "Islanded" | Ticketing honors existing capacity; Growth suspends push stories |
| Enclosure camera failure | Publish `quality: sensor_fault`. Switch to secondary angle | Flag census as "provisional" | Exhibit remains open; keeper scheduled for visual count |
| LLM provider outage | Zero edge impact | Gateway routes to local SLM | Copilot notes lose polish; zero clinical impact |
| Dissolved oxygen crash (< 3 mg/L) | Firmware solenoid valve for backup oxygenation + local siren | Instant `EnclosureUnsafe` projection | Ticketing halts admissions to affected zone |
| Containment breach | Microcontroller solenoid + strobe siren | `ExhibitAvailabilityChanged(closed, safety)` | Security perimeter; turnstiles deny entry to sector |
| Edge gateway failure | VRRP failover to secondary gateway | Detect missing heartbeat; dispatch field engineer | Adjacent zones unaffected |

---

## Domain ADRs

- [ADR-005: Living Collection as Sole Capacity Authority](../adrs/ADR-005-living-collection-as-sole-capacity-authority.md)
- [ADR-007: Strict Tiered Autonomy — AI Assists, Humans Decide](../adrs/ADR-007-tiered-autonomy-ai-assists-humans-decide.md)
- [ADR-009: Edge CV for Piranha Biomass Estimation](../adrs/ADR-009-edge-cv-for-piranha-biomass-estimation.md)
- [ADR-010: Clinical vs. Public Digital Twin Split](../adrs/ADR-010-clinical-vs-public-digital-twin-split.md)
- [ADR-011: Keeper Dispositions as Ground-Truth Training Labels](../adrs/ADR-011-keeper-disposition-as-ground-truth-labels.md)
