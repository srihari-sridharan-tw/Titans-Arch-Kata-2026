# Mermaid Diagram Suite & Notation Legend

## Overview

This directory contains the complete suite of **standalone Mermaid diagrams** for the Von Digitalis Estates digital platform submission. Each diagram file (`.mmd`) is written in standard Mermaid syntax and renders natively on GitHub and within Markdown previewers.

---

## 1. Diagram Inventory

| File Name | Diagram Type | Architectural Focus |
|---|---|---|
| [`c4-01-system-context.mmd`](c4-01-system-context.mmd) | C4 Level 1 | Estate System Context: External actors, primary platform, and SaaS dependencies. |
| [`c4-02-containers.mmd`](c4-02-containers.mmd) | C4 Level 2 | Container View: 4 Modular Monoliths, Kafka event backbone, edge brokers, Model Gateway. |
| [`c4-03-ticketing-components.mmd`](c4-03-ticketing-components.mmd) | C4 Level 3 | Ticketing & Access Components: Ed25519 issuance, offline gate validator, dynamic pricing. |
| [`c4-03-living-collection-components.mmd`](c4-03-living-collection-components.mmd) | C4 Level 3 | Living Collection Components: Edge CV Jetson pipeline, Clinical vs Public Twin, Keeper Copilot. |
| [`c4-03-park-operations-components.mmd`](c4-03-park-operations-components.mmd) | C4 Level 3 | Park Operations Components: Queue sensors, demand forecasting (LightGBM), shift briefing. |
| [`c4-03-guest-growth-components.mmd`](c4-03-guest-growth-components.mmd) | C4 Level 3 | Guest Growth Components: Companion App PWA, Spot Saver client, personalized recommendations. |
| [`edge-fabric-deployment.mmd`](edge-fabric-deployment.mmd) | Deployment Diagram | Physical 55-Zone Edge Fabric: 3-VLAN layout, Jetson nodes, Raspberry Pi CM4 turnstiles. |
| [`model-gateway-flow.mmd`](model-gateway-flow.mmd) | Sequence Diagram | Multi-Model Gateway Execution: Semantic cache, token budgets, 4-tier failover cascade. |
| [`unified-spot-saver-sequence.mmd`](unified-spot-saver-sequence.mmd) | Sequence Diagram | Unified Spot Saver: Pre-visit booking, in-park capacity clamp, Living Collection authority. |

---

## 2. Mermaid Styling & Color Key (`classDef`)

To ensure consistent visual hierarchy across all architectural artifacts, diagrams utilize the following color palette:

```mermaid
graph TD
    classDef person fill:#08427b,stroke:#073b6f,color:#ffffff;
    classDef internalSystem fill:#1168bd,stroke:#0b4884,color:#ffffff;
    classDef externalSystem fill:#999999,stroke:#666666,color:#ffffff;
    classDef edgeFabric fill:#2e7d32,stroke:#1b5e20,color:#ffffff;
    classDef aiSystem fill:#6a1b9a,stroke:#4a148c,color:#ffffff;
    classDef dataStore fill:#f57c00,stroke:#e65100,color:#ffffff;

    P[Person / User Actor]:::person
    S[Internal Estate System]:::internalSystem
    E[External SaaS Dependency]:::externalSystem
    ED[Edge Compute / IoT Device]:::edgeFabric
    AI[AI / Model Gateway / ML Component]:::aiSystem
    DB[(Data Store / Event Backbone)]:::dataStore
```

- **Blue (`#1168bd`)**: Core internal platform containers and microservices.
- **Dark Blue (`#08427b`)**: Human actors (Guests, Keepers, Ride Operators, Executives).
- **Purple (`#6a1b9a`)**: AI models, Model Gateway, LLM-as-a-judge, and ML inference components.
- **Green (`#2e7d32`)**: Physical edge hardware, Jetson nodes, turnstile controllers, and IoT brokers.
- **Orange (`#f57c00`)**: Databases, persistent event backbones (Kafka), and NVMe queues.
- **Gray (`#999999`)**: External third-party systems (Payment Gateways, Weather Radar APIs, Apple/Google Wallets).
