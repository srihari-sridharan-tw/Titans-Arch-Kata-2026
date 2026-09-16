# C4 Architecture Models

## Level 1 — System Context

> Who uses the platform and what external systems does it depend on?

```mermaid
graph TB
    Guest(("👤 Guest<br/>5K–15K/day"))
    Keeper(("🧑‍🔬 Keeper / Vet"))
    RideOp(("🎢 Ride Operator"))
    OpsManager(("📋 Ops Manager"))
    Countess(("👑 Countess"))

    Platform["🏰 Titans Estate Platform<br/><i>Edge-First, AI-Assisted<br/>Heritage Estate Platform</i>"]

    PayGW["💳 Payment Gateway<br/>(Stripe / Adyen)"]
    Weather["🌤️ Weather Service"]
    LLM["🤖 Foundation Model Providers"]
    Push["📱 Push / Email"]
    Social["📸 Social Media API<br/>(Official mentions only)"]

    Guest -->|"Buy tickets, use app, enter gates"| Platform
    Keeper -->|"Rounds, dispositions, clinical notes"| Platform
    RideOp -->|"Dispatch, ride status"| Platform
    OpsManager -->|"Staff deployment, analytics, CapEx"| Platform
    Countess -->|"ROI dashboards"| Platform

    Platform -->|"Charges & refunds"| PayGW
    Platform -->|"Forecasts"| Weather
    Platform -->|"AI tasks"| LLM
    Platform -->|"Notifications"| Push
    Platform <-->|"Consented mentions"| Social

    classDef person fill:#1e293b,stroke:#f59e0b,stroke-width:2px,color:#f8fafc
    classDef system fill:#0f172a,stroke:#0284c7,stroke-width:3px,color:#f8fafc
    classDef external fill:#1e293b,stroke:#64748b,stroke-width:1px,color:#94a3b8

    class Guest,Keeper,RideOp,OpsManager,Countess person
    class Platform system
    class PayGW,Weather,LLM,Push,Social external
```

---

## Level 2 — Container Diagram

> What are the major deployable units and how do they communicate?

```mermaid
graph TB
    subgraph Edge["🏗️ Edge Tier — 55 Zones"]
        direction TB
        MQTT["MQTT 5 Broker Mesh<br/><i>Mosquitto, clustered</i>"]
        NVMe["72h NVMe Buffer<br/><i>Store-and-Forward</i>"]
        IoT["Sensors & Cameras<br/><i>mTLS per device</i>"]
        GScanner["Gate Scanners<br/><i>Ed25519 offline validation</i>"]
        ShedTab["Shed Tablets<br/><i>Keeper UI, local alerts</i>"]
        EdgeInf["Edge Inference<br/><i>Jetson Orin Nano</i>"]

        IoT --> MQTT
        GScanner --> MQTT
        MQTT --> NVMe
        MQTT --> EdgeInf
        EdgeInf --> ShedTab
    end

    subgraph Cloud["☁️ Cloud Tier"]
        direction TB
        Kafka[("Kafka<br/><i>CloudEvents v1.0</i>")]
        
        TK["Ticketing & Access<br/><i>Commerce · Gates · Concierge<br/>Pricing · Credentials</i>"]
        LC["Living Collection<br/><i>Clinical Twin · Public Twin<br/>Edge CV · Husbandry · Telemetry</i>"]
        PO["Park Operations<br/><i>Ride Ops · Queue · Staff Roster<br/>Demand Forecast · Analytics</i>"]
        GG["Guest Growth<br/><i>Companion · Spot Saver<br/>Recommendations · Nudge</i>"]

        MGW["Shared Model Gateway<br/><i>Task routing · 4-tier failover<br/>Cost ceilings · Eval pipeline</i>"]

        AnalyticsDB[("Analytics Store<br/><i>TimescaleDB + DuckDB</i>")]
        ObjectStore[("Object Storage<br/><i>Evidence clips, audit logs</i>")]
    end

    subgraph Guest["📱 Guest Tier"]
        PWA["Companion App PWA<br/><i>Offline-first · LiteDB</i>"]
        Kiosk["Physical Kiosks<br/><i>No-phone parity</i>"]
    end

    MQTT -->|"QoS 1<br/>Store-and-Forward"| Kafka

    Kafka <--> TK
    Kafka <--> LC
    Kafka <--> PO
    Kafka <--> GG

    TK --> MGW
    LC --> MGW
    PO --> MGW
    GG --> MGW

    TK --> AnalyticsDB
    LC --> AnalyticsDB
    PO --> AnalyticsDB

    LC --> ObjectStore

    PWA <--> TK
    PWA <--> GG
    Kiosk <--> TK

    classDef edge fill:#450a0a,stroke:#f97316,stroke-width:2px,color:#f8fafc
    classDef cloud fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    classDef bus fill:#064e3b,stroke:#10b981,stroke-width:3px,color:#f8fafc
    classDef ai fill:#4c1d95,stroke:#a855f7,stroke-width:2px,color:#f8fafc
    classDef store fill:#1e293b,stroke:#94a3b8,stroke-width:1px,color:#94a3b8
    classDef guest fill:#1e293b,stroke:#f59e0b,stroke-width:2px,color:#f8fafc

    class MQTT,NVMe,IoT,GScanner,ShedTab,EdgeInf edge
    class TK,LC,PO,GG cloud
    class Kafka bus
    class MGW ai
    class AnalyticsDB,ObjectStore store
    class PWA,Kiosk guest
```

---

## Level 3 — Component Diagrams (Per Domain)

### Ticketing & Access — Components

```mermaid
graph TB
    subgraph TK["Ticketing & Access (Modular Monolith)"]
        API["API Gateway<br/><i>Rate limiting, DDoS, TLS</i>"]
        Catalog["Catalog Module<br/><i>Products, ticket types</i>"]
        Cart["Cart Module<br/><i>Redis-backed sessions</i>"]
        Pricing["Pricing Engine<br/><i>Rules + AI advisor signal</i>"]
        Commerce["Commerce Module<br/><i>Orders, payments, refunds</i>"]
        Cred["Credential Module<br/><i>Ed25519 JWT issuance</i>"]
        GateVal["Gate Validator<br/><i>Online + offline paths</i>"]
        Concierge["AI Concierge<br/><i>Tool-calling LLM</i>"]
        Capacity["Capacity Module<br/><i>Redis atomic counters</i>"]
        NBA["Next Best Action<br/><i>Post-purchase upsell</i>"]
        Outbox["Transactional Outbox<br/><i>Kafka publisher</i>"]

        API --> Catalog
        API --> Cart
        API --> Commerce
        API --> Concierge
        Cart --> Capacity
        Pricing --> Catalog
        Commerce --> Pricing
        Commerce --> Cred
        Commerce --> Capacity
        Concierge -->|"read-only"| Catalog
        Concierge -->|"read-only"| Capacity
        Commerce --> Outbox
        NBA --> Catalog
        NBA --> Capacity
    end

    DB[("PostgreSQL<br/><i>Orders, tickets, payments</i>")]
    Redis[("Redis<br/><i>Cart, capacity, pricing cache</i>")]
    MGW["Model Gateway"]

    Commerce --> DB
    Cart --> Redis
    Capacity --> Redis
    Concierge --> MGW
    NBA --> MGW
    Pricing -->|"demand signal"| MGW

    classDef mod fill:#1e3a5f,stroke:#38bdf8,stroke-width:1px,color:#f8fafc
    classDef store fill:#1e293b,stroke:#94a3b8,stroke-width:1px,color:#94a3b8
    classDef ai fill:#4c1d95,stroke:#a855f7,stroke-width:2px,color:#f8fafc

    class API,Catalog,Cart,Pricing,Commerce,Cred,GateVal,Concierge,Capacity,NBA,Outbox mod
    class DB,Redis store
    class MGW ai
```

### Living Collection — Components

```mermaid
graph TB
    subgraph LC["Living Collection (Modular Monolith)"]
        Telemetry["Telemetry Ingest<br/><i>MQTT → time-series</i>"]
        Inventory["Inventory Module<br/><i>Species, enclosures</i>"]
        Husbandry["Husbandry Module<br/><i>Feeding, medication, care</i>"]
        ClinicalTwin["Clinical Twin<br/><i>Full vet data</i>"]
        PublicTwin["Public Twin<br/><i>Sanitized projection</i>"]
        Inference["Inference Module<br/><i>Anomaly detection, CV</i>"]
        AlertEngine["Alert Engine<br/><i>Deterministic policy</i>"]
        Copilot["Keeper Copilot<br/><i>GenAI note drafts</i>"]
        Narrator["Story Narrator<br/><i>Guest-facing stories</i>"]
        Outbox["Transactional Outbox<br/><i>Kafka publisher</i>"]
    end

    TimescaleDB[("TimescaleDB<br/><i>Telemetry time-series</i>")]
    PG[("PostgreSQL<br/><i>Inventory, husbandry</i>")]
    ObjStore[("Object Storage<br/><i>Evidence clips</i>")]
    MGW["Model Gateway"]

    Telemetry --> TimescaleDB
    Inventory --> PG
    Husbandry --> PG
    ClinicalTwin --> TimescaleDB
    ClinicalTwin --> PG
    PublicTwin --> ClinicalTwin
    Inference --> TimescaleDB
    AlertEngine --> Inference
    Copilot --> MGW
    Narrator --> MGW
    AlertEngine --> Outbox
    PublicTwin --> Outbox

    classDef mod fill:#064e3b,stroke:#10b981,stroke-width:1px,color:#f8fafc
    classDef store fill:#1e293b,stroke:#94a3b8,stroke-width:1px,color:#94a3b8
    classDef ai fill:#4c1d95,stroke:#a855f7,stroke-width:2px,color:#f8fafc

    class Telemetry,Inventory,Husbandry,ClinicalTwin,PublicTwin,Inference,AlertEngine,Copilot,Narrator,Outbox mod
    class TimescaleDB,PG,ObjStore store
    class MGW ai
```

### Park Operations — Components

```mermaid
graph TB
    subgraph PO["Park Operations (Modular Monolith)"]
        RideOps["Ride Ops Module<br/><i>Dispatch, status, cycles</i>"]
        QueueSense["Queue Sensing<br/><i>Vision counting</i>"]
        StaffRoster["Staff Roster<br/><i>Positions, shifts</i>"]
        DemandForecast["Demand Forecast<br/><i>ML time-series model</i>"]
        Popularity["Popularity Score<br/><i>Weighted composite</i>"]
        StaffRules["Staffing Rules<br/><i>Threshold → recommendation</i>"]
        CapEx["CapEx Flag Engine<br/><i>Sustained trend detection</i>"]
        ShiftBriefing["Shift Briefing<br/><i>GenAI narrative + faithfulness</i>"]
        InvestReport["Investment Report<br/><i>Monthly GenAI narrative</i>"]
        Outbox["Transactional Outbox"]
    end

    TimescaleDB[("TimescaleDB<br/><i>Sensor signals, history</i>")]
    PG[("PostgreSQL<br/><i>Roster, recommendations, audit</i>")]
    MGW["Model Gateway"]

    RideOps --> TimescaleDB
    QueueSense --> TimescaleDB
    DemandForecast --> TimescaleDB
    Popularity --> TimescaleDB
    StaffRoster --> PG
    StaffRules --> PG
    CapEx --> PG
    ShiftBriefing --> MGW
    InvestReport --> MGW
    DemandForecast --> MGW
    StaffRules --> Outbox

    classDef mod fill:#4c1d95,stroke:#a855f7,stroke-width:1px,color:#f8fafc
    classDef store fill:#1e293b,stroke:#94a3b8,stroke-width:1px,color:#94a3b8
    classDef ai fill:#4c1d95,stroke:#a855f7,stroke-width:2px,color:#f8fafc

    class RideOps,QueueSense,StaffRoster,DemandForecast,Popularity,StaffRules,CapEx,ShiftBriefing,InvestReport,Outbox mod
    class TimescaleDB,PG store
    class MGW ai
```

### Guest Growth — Components

```mermaid
graph TB
    subgraph GG["Guest Growth (Modular Monolith)"]
        CompApp["Companion App Backend<br/><i>API for PWA</i>"]
        SpotSaver["Spot Saver Module<br/><i>Virtual queue orchestration</i>"]
        Profile["Guest Profile<br/><i>Consent, preferences, segments</i>"]
        Reco["Recommendation Engine<br/><i>GBDT ranking + GenAI copy</i>"]
        Nudge["Nudge Engine<br/><i>Push, email, in-app</i>"]
        Churn["Churn Prediction<br/><i>Return propensity model</i>"]
        Campaign["Campaign Manager<br/><i>A/B tests, attribution</i>"]
        Outbox["Transactional Outbox"]
    end

    PG[("PostgreSQL<br/><i>Profiles, preferences</i>")]
    Redis[("Redis<br/><i>Session, queue state</i>")]
    MGW["Model Gateway"]

    Profile --> PG
    SpotSaver --> Redis
    Reco --> MGW
    Nudge --> MGW
    Churn --> MGW
    CompApp --> Profile
    CompApp --> SpotSaver
    CompApp --> Reco
    Campaign --> PG
    SpotSaver --> Outbox

    classDef mod fill:#78350f,stroke:#f59e0b,stroke-width:1px,color:#f8fafc
    classDef store fill:#1e293b,stroke:#94a3b8,stroke-width:1px,color:#94a3b8
    classDef ai fill:#4c1d95,stroke:#a855f7,stroke-width:2px,color:#f8fafc

    class CompApp,SpotSaver,Profile,Reco,Nudge,Churn,Campaign,Outbox mod
    class PG,Redis store
    class MGW ai
```
