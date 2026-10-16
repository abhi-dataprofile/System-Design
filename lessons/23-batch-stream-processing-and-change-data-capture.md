# 23 — Batch processing, stream processing, and change data capture

Previous: [Configuration management, policy engines, and tenant customization](22-configuration-management-policy-engines-and-tenant-customization.md). Study time: 50–60 minutes including the exercise.

## Mental model

Batch, streaming, and change data capture (CDC) are different ways to move a changing source of truth into derived state.

```text
source truth → captured changes → durable log/files → transformations → derived views
                    ↑                                   ↓
              position/offset                     checkpoints + sinks
```

- **Batch** processes a bounded dataset: “all invoices closed yesterday.”
- **Streaming** continuously processes an unbounded sequence: “invoice status changes as they occur.”
- **CDC** captures inserts, updates, and deletes from a source database, usually from its transaction log.
- **Materialized views** are derived answers maintained from those inputs.

The core correctness question is not “How fast is it?” It is:

> Can the system explain which source versions produced this result, resume after failure, and replay without losing, duplicating, reordering, or repeating external side effects?

A low-latency pipeline that silently misses deletes is worse than a correct hourly batch for many business workflows.

## When to use each model

Use batch when:

- inputs are naturally bounded, such as daily files or month-end close;
- minutes or hours of freshness are acceptable;
- global recomputation, complex joins, or backfills dominate;
- source systems cannot support continuous capture;
- operational simplicity matters more than latency.

Use streaming when:

- decisions need seconds-to-minutes freshness;
- event-driven automation, alerts, monitoring, or live views matter;
- incremental computation is cheaper than repeated full scans;
- the source produces durable events or CDC;
- consumers can handle duplicates, ordering limits, and replay.

Use CDC when:

- a database is authoritative but does not publish complete domain events;
- downstream search, analytics, caches, or integrations need row-level changes;
- polling would miss rapid updates or impose excessive load;
- deletes and transaction order must be captured.

Use a hybrid when serving needs combine them: an initial snapshot or backfill establishes state, CDC keeps it current, and periodic reconciliation detects drift.

Do not use streaming merely because it sounds modern. A daily regulatory report may be safer and cheaper as a deterministic batch. Do not use CDC as a substitute for domain events when consumers require business intent such as “invoice approved,” not just “status changed from 2 to 3.”

## Start from freshness and correctness

Define:

- required freshness by use case;
- acceptable loss and duplication;
- ordering scope;
- replay and retention window;
- source-of-truth ownership;
- late-data policy;
- deletion behavior;
- privacy and tenant-isolation rules;
- recovery point and recovery time;
- cost and operational ownership.

Example requirements:

| Output | Freshness | Correctness emphasis |
| --- | --- | --- |
| Search index | Minutes | Current version; deleted items absent |
| Executive dashboard | Hourly | Reconciles to ledger totals |
| Safety alert | Seconds | Low loss; duplicate notification controlled |
| AI retrieval corpus | Minutes | Authorized source/version lineage |
| Month-end WIP | Daily/final | Reproducible cutoff and adjustments |
| Clinical operational queue | Seconds–minutes | Patient identity, current status, late corrections |

The same source may feed multiple paths with different service levels.

## Batch architecture

A reliable batch job has:

1. an immutable or versioned input manifest;
2. a deterministic transformation version;
3. a bounded execution window;
4. checkpointed partitions;
5. staging output;
6. validation and reconciliation;
7. atomic publication;
8. a run record with lineage.

Prefer “write new, validate, then swap” over mutating the live table or index in place.

A run manifest can record:

```text
run_id, tenant, input files/partitions, source cutoff,
schema versions, code/config version, row counts,
checksums, output location, validation, publish status
```

Partition by a dimension that supports pruning and replay, such as tenant plus event date. Avoid partitions so small that metadata dominates or so large that one retry repeats hours of work.

## Incremental batch and watermarks

Incremental batch processes only data beyond a stored high-water mark. A timestamp alone is unsafe when records share timestamp precision or arrive late.

Prefer:

- source-provided cursor or log position;
- monotonically increasing version;
- composite watermark such as `(updated_at, stable_id)`;
- overlapping lookback plus deduplication;
- periodic full or sampled reconciliation.

Commit the new watermark only after output is durably published. If the job fails between processing and checkpoint commit, reprocessing should be idempotent.

Late updates older than the watermark require overlap, a correction feed, or reconciliation. “We scan everything newer than midnight” is not enough when an offline system uploads yesterday’s event today.

## Stream architecture

A stream pipeline usually contains:

- producers or CDC connectors;
- a durable broker/log;
- partitions;
- consumer groups;
- stateful processors;
- checkpoint/state storage;
- dead-letter or quarantine handling;
- idempotent sinks;
- schema and lineage metadata.

Partition by the entity whose relative order matters: tenant plus invoice ID, patient encounter, equipment ID, or project. Global order is expensive and usually unnecessary.

More partitions increase parallelism but also coordination, memory, open connections, and rebalancing cost. A “hot” project or tenant can still dominate one partition; use salting only if downstream semantics can restore required grouping.

## Event time, processing time, and watermarks

Three clocks matter:

- **event time:** when the business event occurred;
- **ingestion time:** when the platform received it;
- **processing time:** when a worker handled it.

Use event time for business windows. Use processing time for operational latency.

A watermark estimates that most events before time (t) have arrived. It is not proof that no older event will arrive. Windowing policy must define:

- window type: tumbling, sliding, or session;
- allowed lateness;
- when initial results emit;
- whether late data updates, retracts, or is dropped;
- how consumers receive corrections.

For financial or clinical truth, never silently discard a valid late correction because a window closed. Publish a revised version or route it to reconciliation.

## Delivery semantics

Brokers and stream processors describe message-processing guarantees, but end-to-end behavior includes the sink.

- **At-most-once:** loss possible, no redelivery.
- **At-least-once:** redelivery possible, loss avoided after durable acceptance.
- **Effectively-once:** duplicates occur internally but deterministic IDs and idempotent state transitions make the business result appear once.
- **Exactly-once processing:** possible within bounded transactional systems, but does not automatically cover email, ERP, payment, or other external effects.

Design for at-least-once delivery with:

- stable event IDs;
- deterministic output keys;
- unique constraints or compare-and-set;
- inbox/deduplication records;
- atomic state plus checkpoint where supported;
- idempotent upserts and tombstones;
- reconciliation for outcome-unknown external writes.

Never equate “offset committed” with “business action completed.”

## CDC patterns

### Log-based CDC

Reads the database transaction log or write-ahead log. It usually captures ordered changes with lower source query load and includes deletes.

Benefits:

- low latency;
- transaction ordering and positions;
- before/after values depending on configuration;
- fewer source scans.

Costs:

- database-specific setup and privileges;
- log retention pressure when connectors lag;
- schema-change handling;
- snapshot coordination;
- sensitive columns may be copied unintentionally.

### Polling

Queries rows modified since a cursor. It is simpler but needs a reliable update version, overlap, deduplication, and explicit delete tracking.

### Triggers

Writes change rows into an audit/outbox table. Triggers can capture intent near the transaction but add database coupling and write overhead.

### Transactional outbox

The application writes domain state and an outbox event in the same database transaction. A relay publishes the outbox asynchronously.

Use the outbox when the application owns the write and consumers need a meaningful domain event. CDC can transport the outbox rows without exposing internal table mutations.

## CDC is not a domain model

A row update says what changed physically, not always why.

An invoice status may change because it was approved, reversed, imported, corrected, or repaired. Consumers should not infer business intent from undocumented numeric fields.

Keep raw CDC inside integration boundaries. Map it through an owned, versioned adapter into domain events or canonical records. Protect tenant identity and filter columns before broad distribution.

Avoid coupling dozens of consumers directly to internal tables. A database migration then becomes a company-wide event-schema migration.

## Initial snapshot plus live changes

Bootstrapping CDC requires a consistent handoff:

1. establish or record a source log position;
2. take a consistent snapshot associated with that position;
3. start or retain log capture without gaps;
4. load snapshot rows using source keys and versions;
5. apply changes after the recorded position;
6. deduplicate overlap;
7. validate counts, checksums, and business invariants;
8. publish the derived view.

Implementation details vary by database/connector. The invariant is: every committed source change belongs either in the snapshot or in the captured log, and applying overlap twice must not corrupt state.

During a long snapshot, log retention must be sufficient. If the source discards required log segments, restart from a known consistent point rather than pretending continuity.

## Ordering, transactions, and foreign keys

Ordering guarantees are scoped. A partition may preserve order for one invoice but not across an invoice and its vendor record.

Include:

- event ID;
- tenant;
- entity ID and type;
- source version;
- transaction or commit identifier where available;
- occurred and captured times;
- schema version;
- operation type including delete;
- causation/correlation ID;
- source position.

If downstream state needs multiple tables atomically, group by source transaction where supported or tolerate intermediate states until a transaction-complete marker. Otherwise a child row may arrive before its parent.

Do not rely on broker arrival order as a universal version. Apply entity versions and reject stale updates.

## Stateful processing and materialized views

Stateful operators power joins, windows, counts, and deduplication. State must be:

- partitioned consistently;
- checkpointed durably;
- bounded by retention or compaction;
- migrated across application versions;
- rebuilt from retained input when needed;
- monitored for growth and skew.

A materialized view should state its source cutoff and freshness. Examples:

- open invoices by project;
- current equipment location;
- unresolved clinical operational tasks;
- authorized document chunks by source version.

Derived views are not independent truth. Provide a rebuild path and reconcile them against authoritative sources.

## Sinks and atomic publication

For databases, use idempotent upsert guarded by source version:

```sql
UPDATE invoice_projection
SET status = :status, source_version = :version
WHERE tenant_id = :tenant
  AND invoice_id = :invoice
  AND source_version < :version;
```

Use an insert/upsert form that is atomic in the chosen database; the conceptual guard matters.

For object outputs, write immutable files plus a manifest, then atomically move a catalog pointer.

For search/vector indexes:

- use deterministic document/chunk IDs;
- include tenant, source resource, and source version;
- write a complete new version before deleting the old one when availability matters;
- apply tombstones;
- rebuild into a versioned namespace;
- switch an alias only after validation.

For external APIs, a stream processor should normally create a durable command/workflow, not call the API as an untracked side effect.

## Backfills and replay

Backfills compete with live processing and can reproduce old bugs or side effects.

A safe replay defines:

- source range and immutable input;
- target projection/version;
- transformation and schema versions;
- tenant scope;
- checkpoint;
- resource budget and priority;
- side effects disabled or redirected;
- validation and rollback;
- completion manifest.

Use separate consumer groups or lanes so a historical backfill cannot starve live or reconciliation traffic. Throttle by downstream database, index, model, and vendor quotas.

Replaying facts may rebuild a view. It must not resend email, repay an invoice, or repeat a clinical notification unless explicitly intended and deduplicated.

## Schema evolution

Retain schemas for as long as events may replay. Prefer additive changes and tolerant readers, but do not silently change units, identity, or meaning.

A consumer deployment must handle:

- old events under the retention window;
- current producer schema;
- unknown optional fields;
- explicit null versus absent;
- new enum values;
- tombstones;
- upcasters or version-specific handlers.

When changing semantics, publish a new event/version and run old/new projections side by side. Compare business invariants before switching readers.

## Multi-tenant isolation and fairness

Every event and state key must carry a server-validated tenant identity. Partitioning, storage paths, checkpoints, DLQs, metrics, and replay tools must preserve it.

Controls include:

- tenant-scoped topic or row authorization where necessary;
- encryption and regional routing;
- per-tenant quotas and weighted scheduling;
- bounded message/payload size;
- noisy-neighbor protection;
- tenant-aware lag and freshness metrics;
- replay approval scoped to exact tenants and time ranges.

A tenant filter added only in the final query is too late if raw cross-tenant events already entered an untrusted processor or AI index.

## AI and retrieval pipelines

AI ingestion is a derived-data pipeline:

```text
document/version → parse → extract → chunk → embed → index → retrieve
```

Each stage should carry tenant, source resource/version, parser/chunker/model version, content hash, and deletion state.

When a document changes:

- ignore duplicate content by stable version/hash;
- process the new immutable source version;
- publish chunks atomically;
- ensure retrieval selects only the active authorized version;
- tombstone prior chunks and vectors;
- propagate deletion;
- keep old versions only under approved retention.

Do not stream every small edit directly into a production index without debouncing or version guards. Rapid saves can waste inference capacity and publish intermediate content.

Batch is often better for model evaluation and large re-embedding. Streaming is useful for freshness. A versioned shadow index plus alias switch gives safer migration than mixing embedding spaces.

## Construction example

A contractor needs near-real-time job-cost and WIP views from an ERP, plus searchable project documents.

- The ERP database is authoritative for commitments, invoices, costs, and payments.
- Log-based CDC captures row changes into a restricted integration stream.
- An adapter maps physical rows into versioned invoice and cost events with tenant/project identity.
- Events partition by project and business entity; projections use guarded source versions.
- A transactional outbox from the approval application publishes meaningful “approval granted” events.
- Nightly batch reconciles project totals against an ERP cutoff and reports mismatches.
- Month-end WIP runs from a frozen manifest so late adjustments become explicit corrections rather than silently changing the closed report.
- Document updates trigger a debounced ingestion workflow. Chunks and embeddings use deterministic IDs and versioned namespaces.
- Backfills run in a lower-priority lane and cannot call the ERP or resend notifications.
- An ERP timeout after an invoice command remains an outcome-unknown workflow reconciled by operation ID; replaying CDC does not repeat the command.

## Healthcare operations variation

A healthcare operations platform consumes patient movement, appointment, and discharge events while producing operational queues—not independent clinical truth.

Events carry patient/encounter identifiers, source version, event/effective time, correction status, provenance, and tenant/facility scope. Patient identity resolution occurs in an authorized boundary. Late corrections revise the operational projection; a closed window does not erase them.

Streaming supports timely bed-management or follow-up queues. Batch reconciles census and encounter totals to the authoritative clinical system. PHI is minimized in broker payloads, logs, DLQs, and test fixtures. Retention, regional processing, and access controls apply to both raw streams and state checkpoints.

Clinical orders or medication changes are not reissued by replay. Consequential actions remain approval-bound workflows with current authorization and source-version checks.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Batch | Simple, reproducible, efficient scans | Higher latency |
| Streaming | Fresh incremental results | Stateful operational complexity |
| Log-based CDC | Complete low-latency changes | Source/log coupling |
| Polling | Easy to start | Missed deletes and late-update risk |
| Domain outbox | Captures business intent | Application changes and relay |
| Raw table CDC | Broad coverage | Leaks physical schema |
| Event-time windows | Business-correct aggregation | Late-data complexity |
| Large replay retention | Rebuildability | Storage and privacy cost |
| Exactly-once framework | Fewer internal duplicates | Does not cover all sinks |
| Idempotent at-least-once | Robust across systems | Key/version design needed |
| One global stream | Simple discovery | Isolation and blast-radius risk |
| Per-tenant streams | Strong isolation | Operational overhead |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Timestamp cursor without tie-breaker | Updates skipped | Composite cursor or source position |
| Checkpoint commits before sink | Data loss | Atomic/checkpoint-after durable write |
| Sink succeeds before checkpoint | Duplicate processing | Idempotent versioned sink |
| Snapshot/CDC gap | Permanent missing rows | Position-bound consistent handoff |
| Logs expire during snapshot | CDC continuity lost | Retention sizing or restart |
| CDC treated as domain intent | Brittle wrong semantics | Owned adapter/domain events |
| Partition key ignores entity order | Stale state wins | Entity-scoped key and version guard |
| Late event silently dropped | Wrong financial/clinical view | Correction or reconciliation path |
| Delete not captured | Ghost search/retrieval result | Tombstones and purge verification |
| DLQ contains sensitive payloads | Privacy exposure | Restricted quarantine and redaction |
| Replay invokes external action | Duplicate payment/notification | Side-effect-free projection rebuild |
| Backfill shares live capacity | Freshness SLO collapse | Isolated quotas and lanes |
| Schema removed before retention ends | Replay failure | Retain decoders/upcasters |
| Vector spaces mixed | Invalid retrieval | Versioned namespace and alias |
| Tenant missing from state key | Cross-tenant contamination | Validated tenant-aware keys |
| Dashboard lacks cutoff | Users compare unlike data | Publish freshness and source position |

## What to measure

Track source position, end-to-end freshness, consumer lag by tenant and partition, throughput, duplicate and stale-event rejection, watermark delay, late-event rate, checkpoint duration and failures, state size, partition skew, source log retention headroom, snapshot progress, DLQ/quarantine age, schema-version mix, sink errors, reconciliation mismatches, deletion propagation, backfill resource use, index-version coverage, and cost per processed business record.

Separate transport success from business correctness. “Zero consumer lag” does not prove the projection matches the ledger.

## Interview prompt

> Design the data pipelines for a multi-tenant AI operations platform. Construction customers need near-real-time ERP invoice/job-cost views, nightly reconciliation, month-end WIP, and searchable documents. Healthcare customers need timely operational queues with late corrections. Sources include databases, files, APIs, and application events. The system must support backfills, deletes, schema evolution, tenant isolation, and safe replay.

Cover batch versus streaming choices, CDC/outbox, initial snapshots, ordering, event time and watermarks, delivery semantics, checkpoints, idempotent sinks, materialized views, reconciliation, replay, AI indexing, observability, privacy, and failure recovery.

## Worked answer

**One-line conclusion:** Use batch for bounded reproducible truth, streaming for freshness, and versioned CDC/outbox events to maintain idempotent derived views that can be reconciled and replayed without repeating side effects.

“I would define freshness and correctness per output. ERP tables enter through log-based CDC, but an owned adapter converts physical changes into tenant-scoped canonical records; application-owned business intent uses a transactional outbox. A position-bound consistent snapshot bootstraps each projection, then changes after that position apply with overlap-safe source versions.

Events use stable IDs, schema versions, transaction metadata, event time, and entity partition keys. Consumers checkpoint only with durable sink state and use unique keys or compare-and-set so redelivery is harmless. Event-time windows declare allowed lateness and issue corrections rather than silently dropping valid financial or clinical updates. Nightly batch reconciles counts, checksums, and business totals to source cutoffs; month-end jobs publish immutable manifests.

Search and AI artifacts carry source, tenant, document, parser, chunker, embedding, and deletion versions. Re-embedding builds a shadow namespace and switches after validation. Backfills use isolated capacity, checkpointed ranges, and disabled external side effects. A replay rebuilds projections but never pays an invoice or resends a clinical notification. Dashboards expose lag, freshness, log headroom, late data, DLQ age, reconciliation mismatch, and tenant skew.”

## Practice lab

Design a pipeline where:

- an ERP contains invoices, commitments, costs, and payments;
- the initial snapshot takes six hours while transaction logs continue;
- one project produces 40% of all updates;
- some source rows share the same `updated_at`;
- late change orders revise yesterday’s job-cost totals;
- a consumer crashes after updating PostgreSQL but before committing its offset;
- a document is replaced and must disappear from vector retrieval;
- a 12-month backfill must not starve live work;
- replay must not repeat ERP writes or notifications;
- healthcare events can arrive corrected and out of order.

Specify freshness targets, capture method, snapshot/CDC handoff, event envelope, partition key, checkpoints, idempotency, watermark/late-data policy, materialized views, reconciliation, deletion, backfill isolation, schema compatibility, and monitoring.

## References

- [Apache Kafka — Design](https://kafka.apache.org/documentation/#design)
- [Debezium documentation](https://debezium.io/documentation/)
- [Google Cloud Dataflow — Streaming pipelines](https://cloud.google.com/dataflow/docs/concepts/streaming-pipelines)
- [Apache Beam — Programming Guide: windowing and triggers](https://beam.apache.org/documentation/programming-guide/#windowing)
- [Martin Kleppmann — Turning the database inside-out](https://martin.kleppmann.com/2015/11/05/database-inside-out-at-oredev.html)
