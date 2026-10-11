# 19 — Data contracts, schema evolution, and integration resilience

Previous: [Disaster recovery, backups, and regional continuity](18-disaster-recovery-backups-and-regional-continuity.md). Study time: 50–60 minutes including the exercise.

## Mental model

An integration contract is a shared promise about meaning, shape, timing, identity, security, and failure behavior. A JSON schema proves that `amount` is a number; it does not prove whether it is dollars or cents, gross or net, estimated or approved.

Model every boundary as:

```text
producer truth → versioned contract → transport → consumer interpretation
```

Failures occur when any side changes independently. Resilient integrations make assumptions explicit, evolve additively, detect incompatibility before release, quarantine invalid data, and preserve enough identity and history to replay safely.

> Validate syntax at ingress, preserve source meaning and lineage, translate at owned boundaries, and never let a schema change silently alter business semantics.

## What belongs in a data contract

| Dimension | Questions |
| --- | --- |
| Semantics | What does each field mean? Units, timezone, precision, enum lifecycle? |
| Identity | What is the stable resource/event/tenant ID? Can IDs be reused? |
| Shape | Required, optional, nullable, defaulted, repeated, bounded? |
| Time | Event time, ingestion time, effective time, update ordering? |
| Delivery | At-most/at-least once, ordering scope, replay, retention? |
| Mutation | Snapshot, delta, correction, deletion, tombstone? |
| Security | Classification, tenant binding, authorization, redaction? |
| Reliability | Timeout, retry, rate limit, deduplication, reconciliation? |
| Ownership | Producer, consumers, on-call, deprecation and support window? |
| Quality | Freshness, completeness, uniqueness, accuracy invariants? |

Examples and prose are part of the contract. Machine-readable schema is necessary but insufficient.

## Contract boundaries

| Boundary | Contract focus |
| --- | --- |
| Synchronous API | request/response, errors, idempotency, timeout |
| Event/message | schema, ordering, delivery, replay, evolution |
| Database view/export | snapshot time, keys, nulls, incremental watermark |
| File exchange | naming, manifest, encoding, checksum, atomic publication |
| Webhook | signature, event ID, retries, ordering, acknowledgement |
| AI tool | typed arguments, authorization, side effect, outcome states |
| Retrieval document | tenant, source/version, chunking, deletion, provenance |

Do not expose an internal database schema as the integration contract. Internal normalization and migrations should not force every consumer to change.

## Compatibility

Compatibility has direction:

- **backward compatible:** new consumers can read old data;
- **forward compatible:** old consumers can read new data;
- **full compatible:** both directions;
- **breaking:** coordinated migration or new contract version required.

Usually safe changes:

- add an optional field with defined absence behavior;
- add an enum value only if consumers tolerate unknown values;
- add an endpoint or event type;
- widen a documented range when consumers support it.

Usually unsafe:

- rename/remove a field;
- change units, timezone, sign, precision, or meaning;
- make optional data required;
- reuse an ID with new semantics;
- change ordering or delivery guarantees;
- turn `null` into zero or empty string;
- add enum values to exhaustive old consumers.

“Optional” must define three distinct states when relevant: absent, explicitly null/unknown, and known empty/zero.

## Tolerant readers, strict writers

Producers should emit exactly the documented form. Consumers should ignore unknown optional fields but reject data that violates safety-critical meaning.

Tolerant reading is not “accept anything.” Never coerce:

- invalid tenant/resource identity;
- unsupported currency or units;
- impossible dates;
- unknown action type that could cause a write;
- missing idempotency or version fields for consequential operations.

Preserve the raw source payload in restricted storage when audit/reprocessing requires it, then map into a validated canonical model.

## Canonical model versus point-to-point mapping

A canonical internal model can isolate product logic from vendor-specific APIs:

```text
Procore / ERP / email / FHIR
            ↓ adapters
versioned canonical operations model
            ↓ domain workflows
```

Benefits:

- one semantic vocabulary;
- centralized validation, tenant binding, and lineage;
- easier testing and vendor replacement;
- domain workflows avoid vendor quirks.

Costs:

- lowest-common-denominator design;
- complex mappings and lost vendor detail;
- central-team bottleneck;
- canonical changes affect many adapters.

Keep source-specific extensions and original references rather than forcing every concept into an inaccurate universal field. Own mappings explicitly and test round trips where writes return to the source.

## Schema registry and governance

For each contract store:

- globally unique name and version;
- schema plus semantic documentation;
- owner and supported consumers;
- compatibility mode;
- examples and invariants;
- classification and retention;
- deprecation date and migration guide;
- generated code or validation artifacts;
- change history and approvals.

CI should reject incompatible changes, but policy cannot infer every semantic break. Review changes to units, identity, authorization, lifecycle, or business meaning even when the schema tool says compatible.

## Consumer-driven contract tests

Provider tests prove it meets published contracts. Consumer-driven tests capture the subset each consumer relies on.

Test:

- old producer → new consumer;
- new producer → old consumer;
- unknown fields/enums;
- missing optional fields and explicit null;
- boundary values and precision;
- duplicate/out-of-order events;
- corrections and tombstones;
- rate limits, timeouts, and error bodies;
- tenant mismatch and unauthorized references;
- recorded real vendor examples with sensitive data removed.

Contract tests complement end-to-end tests. They are faster and attribute failures, but cannot prove external vendor behavior has not changed.

## Event evolution

Prefer immutable facts and explicit event types:

```text
InvoiceSubmitted.v1
InvoiceCodingCorrected.v1
InvoiceApprovalRevoked.v1
```

An event envelope should include event ID, type/version, tenant, resource ID/version, occurred time, producer, correlation/causation IDs, and payload.

Consumers deduplicate by event ID and apply guarded resource versions. Partition ordering should match the business entity when order matters. Cross-entity global order is expensive and often unnecessary.

Never silently edit an already-published event. Publish a correction or tombstone. Retain schemas as long as retained events can be replayed.

## API evolution

Use additive evolution within a stable version. Create a new major version when semantics cannot coexist.

For writes:

- require an idempotency key;
- use expected resource version/ETag;
- return a durable operation ID;
- distinguish validation, authorization, conflict, rate-limit, transient, permanent, and outcome-unknown errors;
- document retry rules.

A `200 OK` response with a vendor-specific error string is not a successful contract. Normalize it in the adapter and preserve the raw response reference for diagnosis.

Deprecation requires usage telemetry, consumer inventory, notice, migration tooling, and an enforced sunset. “Nobody told us they used it” is not evidence of zero consumers.

## Webhooks and polling

Webhook receiver:

1. verifies signature, timestamp, and replay window;
2. records event ID and raw bounded payload durably;
3. acknowledges quickly;
4. validates and processes asynchronously;
5. deduplicates retries;
6. handles out-of-order delivery using resource version;
7. fetches current source state when the event is only a notification;
8. quarantines poison payloads with an owner.

Never treat webhook possession as authorization to another tenant’s resource.

Polling is useful when webhooks are incomplete or lossy. Track a durable high-water mark with overlap to catch late updates, then deduplicate. Avoid timestamps alone when updates can share precision; use `(updated_at, stable_id)` or vendor cursors.

## Rate limits and backpressure

Adapters should expose vendor quotas as capacity, not hide them.

Use:

- per-vendor and per-tenant token buckets;
- bounded concurrency;
- `Retry-After` and rate-limit headers;
- exponential backoff with jitter;
- retry budgets and deadlines;
- priority for reconciliation and time-sensitive work;
- circuit breakers;
- queue age and drain-time alerts.

Retries consume quota and must preserve idempotency. A bulk backfill must not starve interactive or reconciliation calls.

## Migration patterns

For a breaking semantic change:

1. define v2 and transformation rules;
2. deploy consumers that read v1 and v2;
3. produce v2 in shadow or dual-publish mode;
4. compare business invariants, not only counts;
5. backfill with checkpointed, idempotent jobs;
6. switch reads by cohort;
7. monitor mismatch and DLQ;
8. stop v1 production after consumer inventory reaches zero;
9. retain v1 decoding through replay/retention window;
10. remove migration code deliberately.

Dual writes can diverge. Prefer one authoritative write plus transactional outbox and independently observable projections. If dual write is unavoidable, define which side wins and how reconciliation works.

## Data quality and quarantine

Rejecting all imperfect data can cause an outage; accepting it can corrupt truth. Classify:

- **fatal:** tenant mismatch, invalid identity, unsupported currency for payment;
- **quarantine:** unknown enum, missing optional business mapping needing review;
- **repairable:** normalized whitespace or documented legacy format;
- **warning:** non-critical field omitted.

Quarantine stores source, contract version, validation errors, owner, retry eligibility, and resolution. Never let a poison message retry forever and block its partition.

Measure freshness, completeness, uniqueness, referential integrity, valid-value rate, duplicate rate, schema-version mix, quarantine age, and reconciliation mismatch.

## AI and retrieval contracts

AI pipelines need versioned contracts for:

- extracted fields with value, confidence, location, and source version;
- chunks with tenant, document/version, offsets, parser/chunker version;
- embeddings with model/dimension and index namespace;
- retrieval results with authorization decision and rank provenance;
- prompts and model structured output;
- tool schemas and outcome states;
- evaluation examples and labels.

Never mix embedding dimensions/spaces because both are arrays of numbers. Do not allow the model to invent tenant IDs, resource IDs, currency, or authorization context. Structured model output is untrusted producer data and goes through normal validation.

## Construction example

An ERP changes invoice amount from decimal dollars to integer cents and adds `REVERSED` to status. A Procore-style webhook may arrive out of order.

The adapter:

1. pins the vendor API/schema version and detects the change in contract tests;
2. preserves raw source amount/unit and maps to canonical money `{currency, minor_units}`;
3. rejects ambiguous historical payloads rather than guessing;
4. introduces canonical `REVERSED` lifecycle semantics and makes consumers tolerate unknown source statuses;
5. uses event ID for deduplication and resource version for ordering;
6. fetches current invoice state when webhook payload is only a notification;
7. dual-publishes canonical v1/v2 while invariant checks compare totals, retainage, and approval status;
8. quarantines unmapped statuses;
9. protects ERP writes with expected version and operation ID;
10. removes v1 only after retained events and consumers have migrated.

## Healthcare variation

An HL7/FHIR integration receives corrected lab results and medication statuses. “Final,” “corrected,” “entered-in-error,” and “cancelled” are semantic states, not arbitrary strings.

The canonical record preserves patient/encounter identity, source system, effective time, issued time, version, correction relationship, coding system, units, and provenance. Patient-scoped authorization is applied before data enters AI context. A late preliminary result cannot overwrite a corrected final result.

Unknown codes are quarantined or shown for clinician review; they are not mapped by an LLM guess. PHI remains protected in raw-payload archives, test fixtures, and contract failure logs.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Canonical model | Isolates domain logic | Mapping and governance cost |
| Point-to-point | Fast first integration | N-squared coupling |
| Additive schema | Smooth coexistence | Old fields accumulate |
| Major version | Clean semantics | Parallel support/migration |
| Strict validation | Protects correctness | Quarantine/availability cost |
| Tolerant reader | Evolution flexibility | Can hide producer defects |
| Dual publish | Safe comparison | Cost and divergence |
| Event replay | Rebuildable state | Side-effect danger |
| Webhook | Low-latency updates | Loss, duplicates, ordering |
| Polling | Reconciliation coverage | Quota and staleness |
| Raw source retention | Reprocessing/audit | Privacy and storage burden |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Schema valid, meaning changed | Silent wrong totals | Semantic review/invariants |
| Field renamed in place | Old consumers break | Additive field/new version |
| Unknown enum crashes consumer | Pipeline outage | Unknown handling/quarantine |
| Null treated as zero | Financial corruption | Explicit absence semantics |
| Internal DB is public contract | Migration couples everyone | Stable boundary model |
| Webhook processed synchronously | Timeout/redelivery storm | Durable accept then async |
| Timestamp-only polling | Missed equal/late updates | Cursor or composite watermark |
| Retry without idempotency | Duplicate action | Stable operation key |
| Dual write without reconciliation | Divergent truth | Authority and mismatch checks |
| Replay invokes tools | Duplicate payment/message | Replay-safe adapters |
| Poison message loops | Partition blocked | DLQ/quarantine |
| Embedding spaces mixed | Invalid retrieval | Versioned index namespace |
| Model output trusted | Tenant/action corruption | Validate and bind server context |
| PHI/raw payload in logs | Privacy incident | Restricted archive/reference |

## What to measure

Track schema-version distribution, compatibility failures, unknown enums/fields, validation and quarantine rates, oldest quarantined item, duplicate/out-of-order events, consumer lag, webhook signature failures, polling watermark age, vendor throttling and retry amplification, mapping mismatches, v1/v2 invariant differences, replay outcomes, external reconciliation, contract owner/SLA, and deprecation usage.

## Interview prompt

> Design resilient integrations for a multi-tenant AI assistant connecting construction documents, Procore-like project data, an ERP, email ingestion, and internal workflows. Vendors change APIs, webhooks duplicate and reorder, old events must replay, and AI retrieval depends on versioned extracted data. Adapt the design for healthcare HL7/FHIR integrations.

Cover semantic contracts, compatibility, canonical models, APIs/events/webhooks/polling, idempotency, ordering, rate limits, validation/quarantine, migration, replay safety, AI lineage, observability, and security.

## Worked answer

**One-line conclusion:** Put a versioned semantic contract and owned adapter at every external boundary, evolve additively, quarantine ambiguity, and make replay reconstruct state without repeating side effects.

“I would keep vendor schemas outside domain workflows. Each adapter validates a pinned source version, tenant/resource identity, units, lifecycle, event ID, resource version, and provenance, then maps into a versioned canonical model while retaining a restricted raw reference. APIs require idempotency and expected versions; webhooks are signature-checked, durably accepted, deduplicated, and processed asynchronously. Polling with an overlapping composite watermark reconciles missing events. Events use immutable envelopes with causation, and corrections/tombstones replace mutation. CI runs schema compatibility and consumer-driven tests across old/new combinations, plus semantic invariants such as invoice totals and status transitions. Breaking changes use readers that accept v1/v2, shadow or dual publication, checkpointed backfill, invariant comparison, cohort switch, and delayed v1 removal. Retries obey vendor quotas and preserve operation IDs. Replay disables external tools and rebuilds projections/indexes only. AI chunks, embeddings, retrieval, prompts, and tool outputs all carry lineage/version contracts and remain untrusted until validated.”

## Practice lab

Design a migration where:

- ERP changes dollars to cents and adds two statuses;
- webhook events duplicate, reorder, and sometimes omit details;
- 5% of projects still use consumer v1;
- 90 days of retained v1 events may replay;
- a new parser changes chunk offsets and embedding model;
- one timeout may have accepted an invoice write.

Define v2 semantics, compatibility strategy, canonical mapping, webhook/poll reconciliation, consumer tests, quarantine rules, dual-publish invariants, replay behavior, index migration, deprecation gates, and external-outcome reconciliation.

## References

- [Confluent — Schema Evolution and Compatibility](https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html)
- [Google Cloud — API design guide: compatibility](https://cloud.google.com/apis/design/compatibility)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [AsyncAPI specification](https://www.asyncapi.com/docs/reference/specification/latest)
- [HL7 FHIR — Resource formats](https://hl7.org/fhir/formats.html)
