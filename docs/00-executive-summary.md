# Executive Summary — Von Digitalis Estates Platform

## For the 72nd Countess Von Digitalis

Your Ladyship,

The Von Digitalis Estates are a national treasure: 40 heritage amusement rides, 200+ exotic animals across 55 enclosures, and a grounds where the WiFi drops before a visitor finishes buying a family pass. The mandate is clear — triple daily visitors from 5,000 to 15,000 in three years, or sell the carnivorous plant collection.

This document presents the architecture for a **digital platform that will make that growth safe, profitable, and sustainable** — without turning the estate into a technology experiment.

---

## What the Platform Does

**Four operational capabilities, running on one coherent estate platform:**

1. **Ticketing & Access** — Online ticket sales (including family passes), dynamic pricing that responds to demand within ±30% human-approved guardrails, an AI Guest Concierge that speaks in natural language but never sets a price, and gate scanners that work flawlessly even when the WiFi is down.

2. **Living Collection** — Continuous monitoring of every animal across 55 enclosures. Computer vision counts jumping piranhas as a school (because you can't tag a piranha). Keepers receive instant alerts for feeding anomalies or containment breaches. The system distinguishes clinical data (for vets) from public data (for guests). A piranha containment breach triggers a physical siren on a microcontroller — no cloud, no AI, no WiFi required.

3. **Park Operations** — Real-time ride operations, queue wait times, crowd heatmaps, and popularity scoring. The system recommends staffing moves based on demand forecasts, but a duty manager always decides. GenAI writes shift briefings and investment reports — but a deterministic faithfulness check ensures every number is traceable to source data before delivery.

4. **Guest Growth** — A companion app (works offline), virtual queuing across rides and enclosures, personalized recommendations, and nudge-based retention campaigns. The app never shows a recommendation that would send a family into an overcrowded zone — it checks the simulation first.

---

## How AI Is Used — And Where It Is Not

AI is deployed deliberately, with clear boundaries:

| AI Does | AI Does Not |
|---|---|
| Recommend ticket bundles in natural language | Set, store, or transmit any price |
| Count piranha via edge computer vision | Decide containment policy |
| Forecast crowd demand for staff recommendations | Automatically move staff |
| Draft clinical keeper notes from telemetry | Override veterinary judgment |
| Generate shift briefings from structured data | Derive any figure it presents |
| Rank recommendations for guests | Override capacity limits |

Every AI decision has a human approval step, a fallback that works without AI, and a cost ceiling that can't be exceeded.

---

## What Makes This Architecture Different

1. **Edge-First, Not Cloud-First.** Animal safety decisions happen on local hardware. If the cloud disappears for 72 hours, the estate still operates — tickets are validated, animals are monitored, rides dispatch safely.

2. **Honest Uncertainty.** When the system can't tell you exactly how many piranhas there are, it says *n̂ = 41, CI [35, 48]* instead of pretending to know. When a demand forecast is stale, the dashboard says "estimated." The architecture never fabricates confidence.

3. **AI Provider Independence.** The estate's AI runs through a shared Model Gateway. If OpenAI triples prices tomorrow, the system fails over to a secondary provider, then to a local model, then to static templates. No code changes. No downtime.

4. **Under $0.04 per visitor per day** at 15,000 scale, including all AI costs.

---

## Investment Summary & Financial Rollup

Figures reflect estate-wide planning envelopes from our consolidated cost analysis:

| Cost Category | 5,000 visitors/day (~150K/mo) | 15,000 visitors/day (~450K/mo) | Dominant Driver |
|---|---|---|---|
| **Cloud Infrastructure OpEx** | $3,380 – $5,550 / mo | $7,050 – $11,500 / mo | Relational DBs, TSDB, Kafka cluster |
| **AI Inference & Model Gateway** | $820 – $1,650 / mo | $1,800 – $3,500 / mo | Concierge chats (cost-fused at $0.15/session) |
| **Total Cloud + AI OpEx** | **$4,200 – $7,200 / mo** | **$8,850 – $15,000 / mo** | Shared infrastructure amortizes |
| **Infrastructure + AI Cost / Visitor** | **$0.033 – $0.053** | **$0.021 – $0.037** | **Drops by ~33% at scale** |
| **Edge Hardware (One-Time CapEx)** | $51,000 – $120,000 | $51,000 – $120,000 | 6 cluster gateway pairs, sensors, scanners |

### What the Shared Spine Is Worth

Four bounded contexts run on one Kafka backbone, one Model Gateway, one evaluation pipeline, and one 6-pair edge fleet. Letting each context own these outright is the obvious alternative, and it carries a price:

| Per-domain infrastructure not required | Cost of that alternative |
|---|---|
| A second operations edge gateway fleet | $8,000 – $18,000 CapEx + $200 – $450/mo |
| A Model Gateway and SLM pool per context | $400 – $900 / mo |
| A separately hosted forecast stack for pricing | $200 – $400 / mo |
| An evaluation pipeline per context | $200 – $500 / mo |
| A messaging substrate per context | $400 – $800 / mo |
| **Total at 15K/day** | **~$1,200 – $3,000 / mo OpEx + $8K – $18K CapEx** |

Payment provider fees (~1.4% + $0.20) remain the largest external variable cost ($8,500–$10,500/mo at 15K/day), but scale strictly with gross revenue.

---

## Timeline

| Phase | Duration | Deliverable |
|---|---|---|
| **Phase 1: Foundation** | Months 1–4 | Edge mesh, Ticketing (online + offline gates), basic Living Collection telemetry |
| **Phase 2: Intelligence** | Months 5–8 | AI Concierge, dynamic pricing, demand forecast, edge CV for piranhas |
| **Phase 3: Growth** | Months 9–12 | Companion app, virtual queuing, recommendation engine, retention campaigns |

---

*The piranhas are counted. The tickets are valid. The WiFi is down. Everything still works.*
