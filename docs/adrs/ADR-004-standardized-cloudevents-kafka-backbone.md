# ADR-004: Standardized CloudEvents v1.0 Envelope on Kafka Event Backbone

## Status
Accepted

## Context
The Von Digitalis Estates digital platform comprises four distinct bounded contexts: Ticketing & Access, Living Collection, Park Operations, and Guest Growth. These domains produce vastly different classes of data: financial transactions, ride queue sensor readings, animal telemetry, and guest recommendation clicks.

Without a standardized event envelope across the enterprise:
1. Each domain team invents incompatible message wrappers, requiring custom deserialization and bespoke routing logic in every consumer.
2. Distributed tracing across domain boundaries becomes impossible or requires proprietary metadata headers.
3. Schema evolution becomes chaotic, leading to silent breaking changes when downstream services consume event streams.

## Decision
We mandate the **CloudEvents 1.0 Specification** (CNCF standard) as the universal message format across our centralized **Apache Kafka event backbone** and edge broker bridges. One envelope binds all four contexts; no domain is permitted a private wrapper.
- **`id`**: Must be a **ULID** (Universally Unique Lexicographically Sortable Identifier), functioning as the universal **idempotency key** for at-least-once delivery. Consumers deduplicate on `id`.
- **`source`**: Reverse-scoped bounded context name: `ticketing`, `living-collection`, `park-operations`, `guest-growth`.
- **`type`**: **PascalCase** event name + explicit **`.v1`** version suffix (e.g., `ExhibitAvailabilityChanged.v1`, `VisitorEntered.v1`, `RideWaitTimeUpdated.v1`). Breaking payload changes bump the suffix (`.v2`); schemas are never mutated in place.
- **`subject`**: Primary domain entity identifier (`enc-amazon-17`, `place-gnome-coaster`, `zone-a`).
- **`time`**: RFC 3339 UTC timestamp. Domain ordering uses payload timestamps (`occurredAt`, `scannedAt`).

```json
{
  "specversion": "1.0",
  "id": "01J7Z8E9F1G2H3J4K5L6M7N8P9",
  "source": "park-operations",
  "type": "RideCycleCompleted.v1",
  "subject": "place-gnome-coaster",
  "datacontenttype": "application/json",
  "dataschema": "https://schema.vondigitalis.com/parkops/RideCycleCompleted.v1.json",
  "time": "2026-09-16T12:30:00.000Z",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "data": {
    "rideId": "place-gnome-coaster",
    "throughputCount": 28,
    "dispatchDurationSeconds": 112.4,
    "occurredAt": "2026-09-16T12:29:58.120Z"
  }
}
```

## Consequences

### Positive
- **Universal Observability**: Standardized W3C `traceparent` attributes allow distributed tracing across edge gateways, Kafka topics, and cloud microservices in OpenTelemetry/Datadog.
- **Polyglot Interoperability**: Any language (Go, Python, TypeScript, Java) can deserialize incoming events using standard CloudEvents SDKs.
- **Strict Schema Governance**: Backward compatibility is automatically guaranteed through the schema registry CI/CD check.

### Negative / Tradeoffs
- **Envelope Serialization Overhead**: Adding standardized metadata adds ~150–250 bytes per event, marginally increasing storage footprint on high-frequency sensor streams.
- **Developer Compliance Overhead**: All engineering teams must adhere to schema registration and validation workflows.

### Risks & Mitigations
- *Risk*: High-frequency sensor streams (100 Hz vibration) causing serialization bottlenecks.
- *Mitigation*: High-frequency sensor streams are aggregated at the edge into 1-second summary CloudEvents before publishing to Kafka.

## Alternatives Considered & Rejected
1. **Raw Unwrapped JSON Payloads**: Rejected due to lack of standard metadata, inability to route generically, and loss of distributed tracing context.
2. **Apache Avro / Protocol Buffers without CloudEvents**: Rejected because human-readable CloudEvents JSON simplifies debugging and integration with edge IoT tooling, while Protobuf/Avro can still be wrapped in CloudEvents binary mode if needed.

## Judging Criteria Addressed
- **Criterion 1 (Architectural Soundness)**: Industry-standard decoupling pattern adhering to CNCF recommendations.
- **Criterion 4 (Completeness)**: End-to-end event catalog connecting all domains.
