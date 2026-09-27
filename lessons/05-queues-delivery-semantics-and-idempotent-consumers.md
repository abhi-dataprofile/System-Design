# 05 — Queues, delivery semantics, and idempotent consumers

Previous: [Caching and invalidation](04-caching-and-invalidation.md). Study time: 40–50 minutes including the exercise.

## Mental model

A queue is a durable waiting room between a producer and one or more consumers. It decouples **when work is accepted** from **when work is completed**, buffers short bursts, and lets consumers recover independently.

It does not remove failure. It moves failure into explicit states:

- accepted but not yet processed;
- delivered but not acknowledged;
- processed locally but uncertain externally;
- retried, delayed, dead-lettered, or manually reconciled.

Assume a message may be delivered more than once unless the complete system proves otherwise. The practical target is therefore:

> **At-least-once delivery plus an idempotent business operation produces effectively-once business outcomes within a defined boundary.**

A broker acknowledgement confirms message handling to the broker. It does not prove that a database transaction committed, an ERP accepted an invoice, or a downstream notification arrived.

## When to use a queue

Use a queue when:

- the caller should not wait for slow or variable work;
- traffic arrives in bursts that consumers can drain later;
- a dependency is temporarily unavailable;
- independent consumers react to the same event;
- work needs bounded retries, replay, or auditability;
- expensive AI ingestion or evaluation can run asynchronously.

Prefer a synchronous call when the caller needs an immediate authoritative result, the operation is fast and reliable, or adding delayed completion would make the product harder without improving reliability.

A queue is not unlimited capacity. If arrival rate exceeds processing rate for long enough, queue age grows until the freshness objective is missed.

## Commands, events, and job state

Be precise about the message:

| Message | Meaning | Example |
| --- | --- | --- |
| Command | A request for one owner to attempt an action | `SubmitInvoiceToErp` |
| Event | A fact that already happened | `InvoiceApproved` |
| Work item | A unit of background computation | `ExtractPagesFromDocument` |

Do not name a request as if success already happened. `InvoiceApproved` should only be emitted after approval commits; a request to approve is `ApproveInvoice`.

For user-visible work, keep durable job state in the application database. The queue transports work; it is usually a poor user-facing source of truth. A status resource might move through:

```text
ACCEPTED -> PROCESSING -> SUCCEEDED
                    \-> RETRYING
                    \-> RECONCILIATION_REQUIRED
                    \-> FAILED
```

Clients poll or subscribe to the status resource instead of holding the original request open.

## Message envelope

Use a versioned envelope that supports routing, deduplication, debugging, and evolution:

```json
{
  "message_id": "msg_01...",
  "message_type": "SubmitInvoiceToErp",
  "schema_version": 2,
  "tenant_id": "tenant_123",
  "operation_id": "op_456",
  "aggregate_id": "invoice_789",
  "aggregate_version": 18,
  "occurred_at": "2026-09-27T12:00:00Z",
  "trace_id": "trace_abc",
  "payload": {
    "invoice_id": "invoice_789"
  }
}
```

Use opaque identifiers rather than sensitive records where possible. A consumer can fetch current authorized data from the source. Encrypt transport and storage, minimize retention, and avoid placing PHI, secrets, or full documents in message attributes and logs.

`message_id` identifies one emitted message. `operation_id` identifies the business intent across retries and possibly across several messages. They are not interchangeable.

## Delivery semantics

| Semantic | What can happen | Appropriate use |
| --- | --- | --- |
| At-most-once | Work may be lost, but the broker will not deliberately redeliver it | Disposable telemetry where loss is acceptable |
| At-least-once | Work should not be lost after durable acceptance, but duplicates can occur | Most business workflows with idempotent consumers |
| Exactly-once | A scoped platform guarantee prevents duplicate effects inside specific boundaries | Useful when its transaction boundary includes every important effect |

“Exactly once” is not a universal property of a queue. A broker can deduplicate publication or atomically write to its own log while an external ERP still receives two requests. Always state the boundary: message log, database, consumer transaction, or end-to-end business effect.

Ordering is also scoped. Global order limits throughput and rarely matches the business need. Prefer ordering per aggregate—such as one invoice or patient encounter—using a partition key. Even then, retries and dead letters can create gaps, so consumers should validate entity version and state transitions.

## Producer reliability: the transactional outbox

A dangerous dual write is:

1. commit invoice approval to the database;
2. publish `InvoiceApproved` to the broker.

A crash between the steps leaves approved data with no event. Publishing first creates the opposite problem: a consumer sees an event for a transaction that later fails.

The transactional outbox solves this by writing the business change and an outbox row in the same database transaction:

```sql
BEGIN;

UPDATE invoices
SET status = 'APPROVED', version = version + 1
WHERE tenant_id = :tenant_id
  AND invoice_id = :invoice_id
  AND status = 'PENDING';

INSERT INTO outbox_events (
  event_id, tenant_id, aggregate_id, aggregate_version,
  event_type, payload, created_at
) VALUES (
  :event_id, :tenant_id, :invoice_id, :new_version,
  'InvoiceApproved', :payload, now()
);

COMMIT;
```

A relay publishes unpublished rows and records progress. The relay can crash after publish but before marking the row, so it can publish twice. The outbox prevents lost events; it does **not** eliminate duplicates. Consumers must still be idempotent.

Monitor the oldest unpublished outbox row, not only row count. Ten recent rows may be healthy; one row stuck for an hour may not be.

## Consumer reliability: acknowledge after durable effect

A typical consumer:

1. receives a message under a visibility timeout or delivery lease;
2. validates envelope and schema;
3. starts a database transaction;
4. checks whether the business operation was already applied;
5. applies the state change and records deduplication atomically;
6. commits;
7. acknowledges the message.

If it acknowledges before commit, a crash can lose work. If it commits before acknowledgement, a crash can cause redelivery. Step 4 makes that redelivery harmless.

The visibility timeout must exceed normal processing time with margin. If it expires too early, another worker may process the same message concurrently. If it is excessively long, failed work recovers slowly. Long jobs can renew their lease, but renewal must be bounded so a wedged worker cannot hide a message forever.

## Idempotency patterns

Idempotency means repeating the same business intent does not create an additional unintended effect. It does not mean “return the same HTTP bytes forever.”

### Inbox or processed-message table

Record the consumer and message identity under a unique constraint:

```sql
CREATE TABLE consumer_inbox (
  tenant_id      uuid NOT NULL,
  consumer_name  text NOT NULL,
  message_id     uuid NOT NULL,
  processed_at   timestamptz NOT NULL,
  result_ref     text,
  PRIMARY KEY (tenant_id, consumer_name, message_id)
);
```

Insert the inbox row and update domain state in one transaction. On a duplicate-key conflict, return the stored result or acknowledge the already completed work.

Message-level deduplication is necessary but may not be sufficient. If two different messages represent the same approval intent, enforce a business key such as `(tenant_id, operation_id)`, and use a guarded state transition such as `PENDING -> APPROVED`.

### Naturally idempotent state transitions

“Set invoice status to approved if currently pending” is safer than “increment approval count.” Use compare-and-set conditions, entity versions, uniqueness constraints, and monotonic states.

### External side effects

A local inbox cannot roll back an ERP call. Pass a stable operation key to the ERP if it supports idempotency. Persist the outbound attempt before sending and record the external reference after success.

If the call times out, the outcome is **unknown**, not failed:

1. do not immediately create a new operation key;
2. query the ERP by idempotency key or external reference;
3. if found, record success locally;
4. if definitely absent, retry the same operation;
5. if the ERP cannot be queried safely, move the operation to `RECONCILIATION_REQUIRED` for controlled review.

Do not tell the user “failed” when the only known fact is that the client did not receive a response.

## Retries, poison messages, and dead letters

Retry only failures that may become successful: transient network errors, throttling, lease loss, or a temporarily unavailable dependency. Do not repeatedly retry invalid schemas, missing required fields, forbidden operations, or permanent business-rule violations.

Use:

- exponential backoff with jitter;
- a maximum attempt count or maximum elapsed time;
- per-dependency concurrency limits;
- a dead-letter queue (DLQ) for exhausted or non-retryable messages;
- an operator workflow that fixes the cause before replay.

A DLQ is not an archive you can ignore. Alert on arrivals, preserve the original message and failure class, and make replay auditable and rate-limited. Replays still require idempotency because some messages may have partially succeeded.

## Backpressure and load shedding

Let:

- λ = arrival rate;
- μ = service rate per consumer;
- c = number of consumers.

The queue drains only while `cμ > λ` over a meaningful interval. Scaling on queue length alone can mislead because message sizes and processing times differ. Track **age of the oldest ready message**, processing latency, and completion rate.

Bound concurrency at downstream systems. Adding workers can reduce queue age while overloading the database, ERP, vector store, or model API. For low-priority work, apply quotas, defer, coalesce superseded jobs, or reject new work before the entire system collapses.

Fairness matters in multi-tenant systems. One tenant’s bulk import should not starve urgent work for every other tenant. Use per-tenant quotas, priority classes with starvation protection, or isolated queues when risk justifies the complexity.

## Schema evolution

Messages outlive deployments. Producers and consumers may run different versions simultaneously.

- Add optional fields before making them required.
- Keep consumers tolerant of unknown fields.
- Version incompatible meanings explicitly.
- Test old producers against new consumers and the reverse.
- Retain handlers or migrate queued messages before removing an old version.
- Do not silently reinterpret a field while keeping the same schema version.

For event history, distinguish the schema version from the aggregate version. One describes message shape; the other describes business-state order.

## Construction example: invoice approval to ERP

A multi-tenant construction assistant proposes an invoice approval. A human approves it.

1. The API authenticates the user, checks tenant membership and approval authority, and accepts an idempotency key.
2. One transaction creates or reuses `approval_operation(operation_id, tenant_id, invoice_id, status)`, performs the guarded approval transition, and inserts an outbox event.
3. The API returns `202 Accepted` with a status URL.
4. An outbox relay publishes `SubmitInvoiceToErp` with tenant, operation, invoice, and version identifiers.
5. A worker inserts an inbox row and an outbound-attempt row.
6. It calls the ERP with the stable `operation_id`.
7. On a confirmed ERP response, it records the external reference and marks the operation `SUCCEEDED`.
8. On timeout, it marks the attempt `OUTCOME_UNKNOWN`, checks the ERP, and retries only with the same key or escalates to reconciliation.
9. Notifications are emitted from another outbox after the durable state change.

The UI shows accepted, processing, succeeded, or reconciliation required. Queue internals such as “message received three times” remain operational detail.

### Healthcare variation

For a lab-result ingestion workflow, the source system’s accession or observation identifier can help form a business deduplication key. Preserve amendments as new versions rather than dropping them as duplicates. Keep PHI out of broker metadata where possible, apply minimum retention, audit access, and ensure a DLQ has protections equivalent to the primary path.

## Failure modes

| Failure | Consequence | Design response |
| --- | --- | --- |
| Database commits; publish is lost | Downstream never runs | Transactional outbox |
| Publish succeeds twice | Duplicate delivery | Inbox plus business idempotency key |
| Worker commits; ack is lost | Redelivery after success | Atomic dedup record and state change |
| Visibility timeout expires mid-job | Concurrent processing | Correct lease, bounded renewal, idempotency |
| ERP accepts; response times out | Unknown external outcome | Stable key, status lookup, reconciliation |
| Poison message retries forever | Cost and queue blockage | Classify errors, bounded retries, DLQ |
| One tenant floods queue | Other tenants starve | Quotas, fair scheduling, isolation |
| Old consumer sees new schema | Parse failure or wrong behavior | Compatible evolution and explicit versions |
| Queue unavailable | New work cannot be published | Outbox accumulation, admission control, alerting |
| Consumers fall behind | Stale results and missed SLA | Oldest-age alert, safe scaling, load shedding |

## What to measure

Measure the workflow end to end:

- accepted-to-start and accepted-to-complete latency;
- ready message count and age of oldest message;
- publish, receive, acknowledgement, and redelivery rates;
- processing duration and visibility-timeout expirations;
- attempts by error class and retry delay;
- duplicate detection and idempotency conflicts;
- DLQ arrivals, age, owner, and replay outcomes;
- unpublished outbox age and relay lag;
- external outcomes: success, definite failure, and unknown;
- throughput and backlog by tenant and priority;
- business completion rate, not merely broker acknowledgements.

Alert on user-impacting age and stuck state, not only infrastructure availability.

## Interview prompt

> Design the asynchronous path for approving construction invoices and submitting them to an ERP. Requests may be repeated, workers may crash, the broker may redeliver, and the ERP can accept a request even when your worker times out. The UI must show an honest status, and one tenant’s import cannot starve others.

Spend five minutes defining the API response, job state, message envelope, producer reliability, consumer transaction, external-effect handling, retries and DLQ, ordering, backpressure, and metrics.

## Worked answer

**One-line conclusion:** Use an outbox for durable publication, at-least-once delivery with transactional deduplication, and explicit reconciliation for uncertain external effects.

“I would accept a client idempotency key and return `202` with an operation resource. In one database transaction, I would guard the invoice’s state transition, create or reuse the operation, and write an outbox row. A relay publishes a versioned message keyed by invoice so ordering is scoped per invoice. The consumer writes its inbox record and local state atomically, then acknowledges only after commit. For the ERP call, it uses the stable operation ID as the ERP idempotency key and persists every attempt. A timeout becomes outcome unknown; the worker queries the ERP and retries the same key only when absence is established, otherwise it sends the item to reconciliation. Transient errors use bounded exponential backoff with jitter; permanent errors go to an owned DLQ. Per-tenant quotas and downstream concurrency limits preserve fairness and protect the ERP. I monitor oldest-message age, outbox lag, redeliveries, unknown outcomes, DLQ age, and end-to-end completion latency.”

## Practice lab

Design tables and state transitions for:

- `approval_operations`;
- `outbox_events`;
- `consumer_inbox`;
- `external_attempts`.

Then trace these crashes:

1. after the invoice transaction commits but before broker publication;
2. after the worker commits but before acknowledgement;
3. after the ERP accepts the request but before returning a response.

For each, state what retries, what unique key prevents duplication, what status the user sees, and who resolves an outcome that remains unknown.

## References

- [AWS SQS Developer Guide — Amazon SQS standard queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html)
- [AWS SQS Developer Guide — Visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Apache Kafka documentation — Message delivery semantics](https://kafka.apache.org/documentation/#semantics)
- [RabbitMQ documentation — Consumer acknowledgements and publisher confirms](https://www.rabbitmq.com/docs/confirms)
- [AWS Builders’ Library — Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
