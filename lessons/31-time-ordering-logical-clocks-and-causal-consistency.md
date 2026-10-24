# 31 — Time, ordering, logical clocks, and causal consistency

Previous: [Service discovery, health checking, and resilient routing](30-service-discovery-health-checking-and-resilient-routing.md). Study time: 50–60 minutes including the exercise.

## Mental model

Distributed systems have no single perfectly observed “now.” Machines pause, networks delay and reorder messages, and clocks drift or jump.

> Use physical time for human meaning and deadlines, monotonic time for elapsed duration, and logical/version order for correctness.

Separate four questions:

1. **When did it occur in the business domain?** Event time.
2. **When did this system observe it?** Ingestion or processing time.
3. **Which operation causally depended on another?** Happens-before order.
4. **Which version is authoritative?** Database/log/ownership order.

Two events can share a timestamp yet have a known causal order. One event can have a later timestamp while actually being an older version because its clock was ahead. A timestamp is data; it is not automatically a concurrency-control protocol.

## When ordering matters

Define ordering when a system handles:

- approval before execution;
- update before delete;
- invoice correction after initial import;
- document version before embedding/index publication;
- leader or shard ownership changes;
- stream processing and late events;
- replicated writes;
- audit reconstruction;
- cache invalidation;
- workflow resume;
- model/tool actions based on a particular evidence version.

Do not demand a global total order unless the invariant truly requires it. Total ordering adds coordination, latency, and availability cost. Most systems need one of these smaller scopes:

| Scope | Example |
| --- | --- |
| Per entity | Versions of one invoice |
| Per aggregate | Project/month cost close |
| Per partition | Events in one queue partition |
| Per workflow | Approval and ERP execution steps |
| Per tenant | Policy/configuration epochs |
| Global | Rare control-plane schema or term order |

State the invariant first: “An ERP action may execute only for the currently approved invoice version” is more useful than “events should be ordered.”

## Three kinds of time

### Wall-clock time

Calendar time such as UTC timestamps is useful for user display, audit, retention, contracts, and correlation across organizations.

It can move unexpectedly because of synchronization corrections, manual changes, virtualization, or leap-second handling. Different hosts have uncertainty.

Use UTC internally, retain source timezone/offset when business meaning needs it, and never assume two client timestamps provide a trustworthy total order.

### Monotonic time

A monotonic clock moves forward within one running process and is appropriate for:

- elapsed duration;
- timeouts;
- backoff;
- lease-renewal scheduling;
- latency measurement.

It usually has no calendar meaning and cannot be compared across machines or process restarts.

### Logical time

Logical clocks order events from observed causality or authoritative serialization rather than clock-of-day.

Use logical versions for optimistic concurrency, fencing, replication progress, stream offsets, and workflow transitions.

## Happens-before and concurrency

Event A happens-before B if:

- both occur in one process and A precedes B;
- A sends a message that B receives;
- or the relationship follows transitively.

If neither A happens-before B nor B happens-before A, they are concurrent. Concurrent does not mean they occurred at the exact same physical instant; it means the system has no causal evidence ordering them.

This distinction prevents a common mistake: silently overwriting one concurrent edit because its wall-clock timestamp is a few milliseconds larger.

## Authoritative sequence numbers

The simplest ordering tool is often a version assigned by the component that owns the invariant:

```text
invoice_id = inv_42
version = 17
status = approved
approved_input_hash = ...
```

A write uses compare-and-swap:

```sql
UPDATE invoices
SET status = :new_status,
    version = version + 1
WHERE tenant_id = :tenant
  AND invoice_id = :id
  AND version = :expected_version;
```

Zero updated rows means the caller used stale state.

Useful ordered identifiers include:

- row version;
- database commit position;
- log partition offset;
- workflow revision;
- configuration epoch;
- leader term or fencing token;
- shard placement epoch.

They are ordered only in their declared scope. Offset 500 in partition A cannot be compared with offset 700 in partition B as a global business order.

## Lamport clocks

Each participant maintains a counter:

1. increment before a local event;
2. include the value in messages;
3. on receive, set local value to `max(local, received) + 1`.

If A happens-before B, then Lamport(A) < Lamport(B). The reverse is not guaranteed: a smaller value does not prove a causal relationship.

A process ID can break ties to create a deterministic total order, but that artificial order does not mean one event caused the other.

Use Lamport clocks when you need causality-respecting order without identifying concurrency precisely.

## Vector clocks and version vectors

A vector records the latest counter observed from each participant or replica.

Vectors can determine:

- A causally precedes B;
- B causally precedes A;
- A and B are concurrent.

They are useful for replicated objects and conflict detection. Their cost grows with participants and membership changes, so production systems often use version vectors per replica group, dotted versions, or application-specific conflict metadata.

Detecting a conflict is not resolving it. Resolution may require:

- domain merge;
- preserving both versions;
- human review;
- authoritative source priority;
- retry against current state.

Do not use last-write-wins for money, approval, medication, or other consequential state merely because merging is inconvenient.

## Hybrid logical clocks

A hybrid logical clock combines a physical-time component with a logical counter. It stays close to wall time while preserving a causal order when clocks collide or move backward within assumptions.

Benefits:

- compact sortable timestamps;
- useful for distributed databases and change streams;
- easier operational interpretation than pure logical counters.

Limits:

- still depends on bounded clock behavior for some guarantees;
- does not by itself enforce an application invariant;
- concurrent events can receive a deterministic order without a business reason.

Treat HLC as metadata supporting a consistency protocol, not as permission to overwrite newer domain state.

## Event, processing, and ingestion time

For a concrete delivery record:

- **event time:** when the delivery occurred at the job site;
- **source-record time:** when the source system recorded it;
- **ingestion time:** when your platform received it;
- **processing time:** when a worker transformed it;
- **publication time:** when a derived view became visible.

Retain these separately. A mobile device may upload yesterday’s inspection today. Sorting by ingestion time would misstate operations; accepting event time blindly would let a bad client backdate records.

Store provenance, source ID, source version, and clock/offset metadata. Define validation and correction rules.

## Streams, watermarks, and late data

Partitioned logs usually guarantee order only within a partition. Choose a key so events that need order share a partition, such as `tenant_id + invoice_id`.

A watermark estimates that most events earlier than time T have arrived. It lets a processor close windows despite unbounded delay.

Watermarks are policy, not truth. Define:

- allowed lateness;
- whether late events update prior results;
- correction/retraction events;
- state retention;
- consumer-visible versioning;
- audit history.

For project cost totals, a late change order should create a revised version, not silently mutate a supposedly final number.

## Causal consistency

Causal consistency ensures that if B depends on A, a reader does not observe B without A. Concurrent operations may appear in different orders.

Examples:

- after a user uploads a document, their next search should not claim processing is complete while hiding the upload record;
- an approval screen should not show “executed” before the approval that authorized it;
- a policy decision must identify the configuration version it evaluated.

Implementations may use session tokens, dependency versions, replica progress, or sticky routing. Common session guarantees include:

- read-your-writes;
- monotonic reads;
- monotonic writes;
- writes-follow-reads.

Causal consistency is weaker and often more available than global serializability. It is not sufficient when two concurrent writes must satisfy one exact constraint; use an authoritative transaction or compare-and-swap there.

## Last-write-wins and clock skew

Last-write-wins chooses the value with the largest timestamp. It is simple but can lose valid concurrent updates and reward the most inaccurate clock.

Example:

1. superintendent approves invoice version 12;
2. correction service creates version 13;
3. an offline device with a clock five minutes ahead uploads an edit based on version 11;
4. timestamp-based LWW replaces version 13.

The correct design validates the expected domain version, preserves the stale edit for review, and does not let clock time bypass approval.

LWW may be acceptable for replaceable preferences such as UI theme, when losing a concurrent value is explicitly harmless.

## Ordering at API and event boundaries

Carry stable metadata:

```json
{
  "operation_id": "op_123",
  "entity_id": "invoice_42",
  "entity_version": 17,
  "event_id": "evt_...",
  "event_type": "invoice.approved",
  "occurred_at": "...",
  "recorded_at": "...",
  "producer": "approval-service",
  "schema_version": 3,
  "causation_id": "evt_prior",
  "correlation_id": "workflow_..."
}
```

Use:

- `event_id` for deduplication;
- `operation_id` for idempotent intent;
- `entity_version` for stale-write rejection;
- `causation_id` for causal chain;
- `correlation_id` for workflow tracing;
- timestamps for human and operational context.

None alone guarantees exactly-once delivery.

Consumers should reject impossible transitions, ignore already-applied versions, detect gaps, and fetch/reconcile authoritative state rather than guessing.

## Multi-region ordering

Cross-region physical timestamps are unsafe as the sole writer-selection rule.

Choose:

- one authoritative write region per key;
- consensus/serializable database;
- region-scoped ownership epoch;
- conflict-detecting multi-writer replication;
- commutative data type where valid.

During failover, issue a higher term/epoch and fence the old writer. Preserve the last committed database/log position. A promoted region that is behind cannot make missing operations real by assigning them newer timestamps.

Read-your-write may require routing to a replica that has applied at least the session’s version. If no replica meets the token, wait within the deadline, route to authority, or return a truthful pending response.

## Timeouts, leases, and expiration

Use monotonic elapsed time for local timeouts. Use an authoritative service for shared lease or token expiration.

Business expiry differs from technical timeout:

- API deadline: caller no longer waits;
- lease expiry: ownership may transfer;
- approval expiry: business authorization is no longer valid;
- retention expiry: data becomes eligible for deletion;
- cache TTL: value requires refresh.

A timed-out request may still have succeeded. A lease expiring does not stop the old process. An approval passing its deadline does not automatically undo an ERP side effect. Model these as distinct states with fencing and reconciliation.

## Audit and reconstruction

An audit record needs more than a timestamp:

- actor and authenticated identity;
- tenant and scope;
- entity and before/after versions;
- operation/event/causation IDs;
- source and schema version;
- policy/configuration/model version;
- recorded timestamp and source event time;
- outcome, including unknown;
- immutable integrity controls and retention.

Audit order is often partial. Presenting a single polished timeline without uncertainty can be misleading. Preserve per-source order, clock uncertainty where known, ingestion delays, and causal links.

## AI workflow ordering

AI pipelines create many derived versions:

```text
source document version
  → parser version
    → chunker version
      → embedding model version
        → index publication epoch
          → retrieval evidence set
            → prompt/model/tool versions
```

A generated answer or proposed action should record this lineage. If the source changes after approval, compare the approved input versions before execution.

Streaming model tokens are not durable workflow commits. Tool execution begins only after structured validation, policy checks, approval binding, and an atomic workflow transition.

Evaluation data also needs event and observation time. A label created after an incident should not be attributed to the model version merely because both share a calendar day.

## Construction operations example

A platform ingests invoices from email, Procore, and ERP. A superintendent approves an invoice, a corrected change order arrives late, the approval worker retries, and ERP timestamps use another timezone.

Design:

1. give each invoice one authoritative row version;
2. require approval against an exact invoice and evidence version;
3. emit approval with entity version, event ID, causation ID, and operation ID;
4. partition the log by tenant/invoice to preserve local order;
5. consumers apply only the next valid transition or reconcile a gap;
6. a late change order creates a new invoice evidence version and invalidates stale approval according to policy;
7. execution compare-and-swaps `approved -> executing` for the approved version;
8. ERP requests reuse the operation ID;
9. timeout creates `outcome_unknown`, not “failed”;
10. reconciliation compares ERP reference/version, not local clock order;
11. reports retain business event time, source-record time, and ingestion time;
12. derived job-cost snapshots include source watermarks and publication versions.

This prevents an ahead-of-time email server or late sync from overriding the business state machine.

## Healthcare operations variation

A healthcare operations platform receives FHIR updates, scheduling events, and delayed interface messages. It preserves source identifiers, FHIR resource versions, event time, receipt time, and provenance.

Patient-facing or operational views use read-your-write/session versions where appropriate. Consequential updates use authoritative resource versions and conditional operations; a later client timestamp cannot overwrite a newer clinical record. Concurrent clinically relevant changes are surfaced through the approved workflow rather than silently merged by LWW.

Derived census or scheduling views publish source watermarks and freshness. Late messages generate corrections. PHI is excluded from ordering keys, traces, and broad metrics where not required. Causal metadata supports reconstruction but never replaces clinical authorization or source-system authority.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Wall-clock timestamp | Human meaning | Skew and jumps |
| Monotonic clock | Reliable local duration | Not comparable across hosts |
| Database version | Strong entity order | Central serialization |
| Lamport clock | Causality-respecting order | Cannot detect concurrency |
| Vector clock | Detects concurrency | Metadata and membership cost |
| Hybrid logical clock | Sortable, near physical time | Not an invariant by itself |
| Per-key order | Scalable and sufficient often | No global order |
| Global total order | Simple downstream view | Coordination and latency |
| Last-write-wins | Easy merge | Silent data loss |
| Conflict preservation | No silent overwrite | Application/user complexity |
| Strict late-data cutoff | Bounded state | Missed corrections |
| Revisable windows | Better accuracy | Versioned outputs required |
| Sticky session | Easy read-your-write | Failover/load imbalance |
| Version token routing | Explicit freshness | Client/protocol complexity |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Client time orders writes | Clock-skew data loss | Authoritative version/CAS |
| Wall clock measures timeout | Negative or stretched duration | Monotonic clock |
| Offset treated as global | False cross-partition order | Declare ordering scope |
| Event time equals ingestion time | Late data misrepresented | Store both |
| LWW resolves consequential conflict | Valid update disappears | Domain merge/review |
| Duplicate has new timestamp | Same action applied twice | Stable event/operation ID |
| Consumer sees version gap | Derived state incomplete | Pause/fetch/reconcile |
| Delete arrives before update | Data resurrects | Versioned tombstone |
| Replica read goes backward | Confusing user state | Session/monotonic-read token |
| Failover uses newer timestamp | Stale region overwrites state | Term/epoch fencing |
| Watermark treated as certainty | Late correction lost | Lateness and revision policy |
| Approval not bound to version | Changed input executes | Input hash/version revalidation |
| Stream token triggers tool | Partial output causes effect | Durable validated transition |
| Clock correction breaks lease | Split ownership | Authoritative expiry and fencing |
| Audit sorts only by timestamp | False narrative | Causal IDs and source order |

## What to measure

Track clock synchronization offset and uncertainty; timestamp regressions; compare-and-swap conflicts; stale-version rejections; duplicate event/operation IDs; event-time-to-ingestion lag; out-of-order rate; version gaps; watermark delay; late-event corrections; replica/session wait; read-version regressions; conflict rate and resolution age; failover term changes; rejected stale epochs; approval invalidations; outcome-unknown age; and lineage completeness.

Alert when clock offset exceeds the assumption used by a protocol, consumers accumulate gaps, a replica serves below a required session version, or a consequential action references a stale input version.

## Interview prompt

> Design time and ordering semantics for a multi-tenant AI operations platform. Invoice events arrive from email, Procore, and ERP with skewed clocks; consumers run across partitions; users require read-your-write; corrections arrive late; approvals must bind to exact evidence; a regional failover can promote a lagging replica; and an ERP request can time out after succeeding.

Cover clock types, ordering scope, logical versions, causality, late data, conflict handling, session guarantees, failover fencing, AI lineage, audit, and healthcare adaptation.

## Worked answer

**One-line conclusion:** Use timestamps to describe business and observation time, but use scoped authoritative versions, causal metadata, and fenced ownership to decide what may overwrite or execute.

“I would define order per invoice and workflow rather than impose a global total order. The authoritative database assigns an entity version, and writes use compare-and-swap. Events carry event ID, operation ID, entity version, source time, recorded time, causation ID, and correlation ID. The log is partitioned by tenant/invoice so relevant transitions retain order.

Consumers deduplicate event IDs, reject stale versions, detect gaps, and reconcile from the source of truth. Event time drives business windows, while watermarks and an explicit lateness policy allow corrections to publish revised results. Concurrent consequential edits are preserved for domain resolution, not collapsed with last-write-wins.

A user session carries the minimum version it has observed. Reads route to a replica at or beyond that version, wait within a deadline, or return pending. Approval records bind to the invoice/evidence versions and input hash; execution revalidates them before an atomic state transition.

Regional failover uses a higher ownership term and fencing. A lagging replica cannot repair missing writes with a newer wall-clock timestamp. ERP calls reuse a stable operation ID; timeout becomes outcome unknown and is reconciled against the external reference.

Physical UTC timestamps support audit and display, monotonic clocks support local deadlines, and logical/database versions protect correctness. AI answers and actions record document, parser, index, prompt, model, policy, and tool lineage.”

## Practice lab

An invoice version 12 is approved. A device clock five minutes ahead submits an edit based on version 11. A change order for the same invoice arrives two hours late. The approval event and correction are processed in different partitions. The primary region then fails with its replica one event behind, and the ERP call has an unknown outcome.

Specify:

- authoritative ordering scopes;
- wall, monotonic, event, and ingestion time usage;
- event schema and partition key;
- stale-write and gap behavior;
- late-data correction policy;
- read-your-write token;
- approval invalidation and execution transition;
- regional term/fencing;
- ERP idempotency/reconciliation;
- AI lineage and audit reconstruction.

Then adapt the design to delayed FHIR updates without using client timestamps to overwrite newer clinical state.

## References

- [Leslie Lamport — Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)
- [Martin Kleppmann — Designing Data-Intensive Applications, distributed systems material](https://dataintensive.net/)
- [Google Cloud Spanner — TrueTime and external consistency](https://cloud.google.com/spanner/docs/true-time-external-consistency)
- [Apache Flink — Timely stream processing](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/time/)
- [RFC 3339 — Date and Time on the Internet](https://www.rfc-editor.org/rfc/rfc3339)
