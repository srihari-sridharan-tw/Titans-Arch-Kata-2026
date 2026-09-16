# Architectural Decision Records (ADRs) Index Matrix

## Overview

This directory contains the **16 Architectural Decision Records (ADRs)** defining the foundational design choices, tradeoffs, and governance frameworks of the Von Digitalis Estates Platform. 

All ADRs follow the **MADR (Markdown Architectural Decision Records)** format, providing rigorous technical context, formal decisions, positive and negative consequences, alternatives considered and rejected, and direct mapping to the Architectural Katas 2026 judging criteria.

---

## 1. Comprehensive ADR Mapping Matrix

| ADR # | Title | Primary Domain | Core Architectural Tradeoff | Judging Criteria Addressed |
|---|---|---|---|---|
| [ADR-001](ADR-001-edge-first-mqtt-broker-mesh.md) | Edge-First MQTT 5 Broker Mesh with 72h NVMe Store-and-Forward | Architecture / Edge | Edge buffering vs Cloud-only ingestion | Deployability, Architecture |
| [ADR-002](ADR-002-shared-task-based-model-gateway.md) | Shared Estate-Wide Task-Based Model Gateway with Domain Namespaces | Cross-Cutting AI | Shared Gateway vs Isolated Domain Gateways | Completeness, Deployability |
| [ADR-003](ADR-003-offline-cryptographic-ticket-validation.md) | Offline Cryptographic Gate Validation via Ed25519 & Local Bloom Filters | Ticketing & Access | Local crypto validation vs Central online verification | Architecture, Feasibility |
| [ADR-004](ADR-004-standardized-cloudevents-kafka-backbone.md) | Standardized CloudEvents v1.0 Envelope on Kafka Event Backbone | Cross-Cutting Data | Standard envelope vs Proprietary event schemas | Architecture, Completeness |
| [ADR-005](ADR-005-living-collection-as-sole-capacity-authority.md) | Living Collection as Sole Authority for Exhibit Capacity | Cross-Domain | Animal welfare priority vs Guest queuing optimization | Soundness, Completeness |
| [ADR-006](ADR-006-unified-demand-forecasting-in-park-ops.md) | Unified Multi-Signal Demand Forecasting in Park Operations | Park Operations | Consolidated domain forecasting vs Fragmented analytics | Architecture, Soundness |
| [ADR-007](ADR-007-tiered-autonomy-ai-assists-humans-decide.md) | Strict Tiered Autonomy — AI Assists, Humans Decide | Cross-Cutting AI | Advisory AI vs Autonomous execution | AI Practicality, Ethics |
| [ADR-008](ADR-008-numbers-first-narrative-second-reporting.md) | Numbers-First, Narrative-Second Pattern for Operations & Financials | Park Operations | Deterministic raw data vs Unchecked GenAI summaries | AI Practicality, Soundness |
| [ADR-009](ADR-009-edge-cv-for-piranha-biomass-estimation.md) | Edge CV (YOLO-nano) for Piranha Biomass & Anomaly Detection | Living Collection | Local edge inference vs Cloud video streaming | Edge, Feasibility, Cost |
| [ADR-010](ADR-010-clinical-vs-public-digital-twin-split.md) | Dual-Schema Separation: Clinical Twin vs. Public Twin | Living Collection | Security & clinical isolation vs Unified database | Security, Architecture |
| [ADR-011](ADR-011-keeper-disposition-as-ground-truth-labels.md) | Keeper Dispositions as Ground-Truth Training Labels | Living Collection | Human-in-the-loop validation vs Unsupervised model updates | AI Practicality, Feasibility |
| [ADR-012](ADR-012-unified-ai-eval-pipeline-and-golden-sets.md) | Unified AI Eval Pipeline & Golden Sets | Cross-Cutting AI | Regression-tested golden sets vs Ad-hoc prompt engineering | Soundness, Deployability |
| [ADR-013](ADR-013-modular-monolith-per-bounded-context.md) | Modular Monolith per Bounded Context | Architecture | Modular monolith vs Microservices sprawl | Architecture, Deployability |
| [ADR-014](ADR-014-dynamic-pricing-hard-guardrails.md) | Dynamic Pricing Engine with Hard Guardrails (±30% Cap) | Ticketing & Access | Algorithmic revenue vs Brand reputation & fairness | Ethics, Soundness |
| [ADR-015](ADR-015-deterministic-faithfulness-verification.md) | Deterministic Faithfulness Check Layer for LLM Outputs | Cross-Cutting AI | Automated verification vs Blind trust in LLMs | AI Practicality, Soundness |
| [ADR-016](ADR-016-three-vlan-segmentation-and-mtls-identity.md) | Strict Network Segmentation (VLAN 10/20, 30, 50) & mTLS Device Identity | Security / Network | Physical network segmentation vs Flat network | Security, Feasibility |


