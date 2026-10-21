# 28 — Distributed locks, leases, fencing tokens, and leader election

Previous: [Distributed rate limiting, quotas, and admission control](27-distributed-rate-limiting-quotas-and-admission-control.md). Study time: 50–60 minutes including the exercise.

## Mental model

A lock answers **who may try now**. A lease says **until when ownership is valid**. A fencing token lets the protected resource reject a **former owner that wakes up late**. Leader election selects one coordinator, but does not by itself make that coordinator's actions safe.

> Lease expiry transfers permission; fencing prevents an expired owner from continuing to cause effects.

Distributed systems cannot instantly distinguish a crashed process from one paused by garbage collection, CPU starvation, or a network partition. Consider:

1. Worker A acquires a lease and token 41.
2. A pauses for longer than the lease.
3. Worker B acquires the lease and token 42, then starts work.
4. A resumes. It still has credentials and an old local belief that it owns the job.
5. Unless the protected resource rejects token 41, A can overwrite B or repeat an external action.

The lock service did its job; the resource still needs a stale-owner defense.

Keep these concepts separate:

| Mechanism | Purpose | Important limit |
| --- | --- | --- |
| In-process mutex | Coordinate threads in one process | No protection across processes or crashes |
| Database constraint/CAS | Preserve a data invariant atomically | May require retry under contention |
| Distributed lock | Serialize contenders across processes | A holder can pause or lose connectivity |
| Lease | Time-bound ownership with renewal | Expiration does not stop old code |
| Fencing token | Reject operations from older owners | The resource must compare and enforce it |
| Leader election | Choose one active coordinator | Leadership can change while old work continues |

## First question: can you avoid a distributed lock?

Prefer a data invariant over a coordination protocol when possible:

- unique constraint for "one invoice posting per operation ID";
- compare-and-swap on a version column;
- idempotency key for retried side effects;
- atomic state transition such as `approved -> executing`;
- partition ownership so one worker naturally owns each key;
- commutative updates or an append-only ledger;
- queue semantics with idempotent consumers.

Locks are justified when a non-atomic sequence truly needs temporary exclusive ownership: a singleton scheduler, shard rebalancer, schema migrator, index alias swap, or workflow whose underlying resource cannot perform the invariant atomically.

A lock is never a substitute for idempotency. A response can be lost after a successful side effect, and a later owner may need to retry or reconcile it.

## Safety and liveness

State both goals:

- **Safety:** at most one valid owner may mutate the protected invariant at a time, and stale owners are rejected.
- **Liveness:** if an owner fails, another can eventually take over.

A lock with no expiration can preserve safety but block forever after a crash. A short lease improves recovery but increases false takeover during pauses. A lease without fencing improves liveness while leaving a safety hole.

Define the invariant precisely. "Only one worker runs" is usually too broad. Better: "Only the holder of the newest token may publish the materialized project-cost snapshot for tenant T and period P."

## Lease lifecycle

A lease record commonly contains:

```text
resource_key
owner_id
lease_id
fencing_token
expires_at
version
```

Acquisition must be atomic: create the lease if absent or replace an expired lease while incrementing a monotonically increasing token. Renewal must prove the same owner and lease ID still holds the current version. Release is conditional too; an old owner must not delete a successor's lease.

The worker should:

1. acquire a lease and fencing token;
2. record both in workflow state;
3. start work only after acquisition succeeds;
4. renew well before expiry;
5. stop admitting new sub-work if renewal becomes uncertain;
6. pass the token to every protected write;
7. treat rejection as loss of ownership;
8. release conditionally after work ends.

Use a random lease ID to distinguish repeated acquisitions by the same process. Use monotonic elapsed time inside a process for renewal scheduling. Let the authoritative store determine expiration; do not make safety depend on synchronized application clocks.

A useful timing budget includes lease duration, renewal interval, worst credible store latency, network jitter, scheduling delay, and pause time. Do not choose a five-second lease merely because normal renewal takes 20 ms.

## Fencing tokens

A fencing token is a value that increases with each successful acquisition. The protected store remembers the largest accepted token:

```sql
UPDATE snapshots
SET payload = :payload,
    last_fencing_token = :token
WHERE snapshot_key = :key
  AND last_fencing_token < :token;
```

If the affected-row count is zero, the writer is stale. Equivalent enforcement can live in an object metadata service, message publisher, storage gateway, or downstream command ledger.

Requirements:

- tokens are strictly ordered for the protected scope;
- each takeover obtains a newer token;
- every protected mutation carries the token;
- the resource compares it atomically with the mutation;
- stale rejection is visible and audited.

A timestamp generated by clients is not a reliable fencing token. A random UUID is unique but not ordered. A database sequence, consensus-store revision, or conditional increment can provide ordering.

If a third-party ERP cannot accept a fencing token, place a controlled adapter in front of it. The adapter serializes commands through an authoritative operation ledger, uses vendor idempotency/version support where available, and reconciles outcome-unknown calls. If no component can reject or deduplicate a stale command, the system cannot promise strict single execution; say so and constrain the workflow.

## Leader election

Leader election is useful for singleton coordination:

- scheduling periodic reconciliation;
- assigning partitions to workers;
- running a schema migration;
- selecting one regional control-plane writer;
- coordinating a failover.

A practical design uses a consensus-backed service or transactional database record:

1. candidates attempt conditional acquisition;
2. the winner receives a lease and term/token;
3. followers observe the current term;
4. the leader renews;
5. on expiry or explicit resignation, a candidate obtains a higher term;
6. every leader-issued write includes that term.

Keep the leader's job small. Let it assign durable, idempotent tasks rather than doing all work itself. That reduces failover time and blast radius.

Leader election does not guarantee:

- exactly-once task execution;
- completion before lease expiry;
- that the old leader immediately stops;
- that external systems honor leadership;
- progress during loss of quorum.

Prefer one leader per shard or bounded domain over one global leader when independent work can proceed separately.

## Coordination-store choices

### Transactional database

A row with conditional updates, a monotonically increasing version, and server-side time can work well when the database already owns the invariant.

Benefits: fewer systems and atomic coordination with related data. Costs: contention, connection coupling, and careful transaction semantics. PostgreSQL advisory locks are session- or transaction-scoped and useful for cooperative database clients, but they do not create fencing for arbitrary external effects.

### Consensus-backed coordinator

Systems such as etcd or ZooKeeper provide ordered state changes, ephemeral/session semantics, and watch mechanisms.

Benefits: well-defined coordination primitives and terms/revisions. Costs: another critical control plane, quorum requirements, operational complexity, and watch/reconnect edge cases.

Treat watches as notifications, not the sole source of truth. After reconnect, re-read authoritative state.

### Cache or key-value store

Atomic conditional creation plus expiry can implement a useful best-effort lock. But failover, replication lag, clock assumptions, and pause behavior must match the required safety.

Even a correctly acquired cache lock needs fencing when stale effects matter. Do not infer strict safety from the word "distributed."

## Granularity, ordering, and contention

Choose the narrowest key that protects the invariant:

```text
tenant/{tenant_id}/project/{project_id}/period/{month}/cost-close
```

A global lock is simple but destroys parallelism. A key that is too narrow permits conflicting work.

When an operation needs multiple locks:

- define a global acquisition order;
- avoid holding a lock during slow user input or remote calls;
- use bounded wait and cancellation;
- release conditionally;
- detect and measure contention.

Lock ordering prevents many deadlocks. Timeouts restore liveness but do not repair partially completed side effects. Use workflow state and compensation/reconciliation for that.

Fairness is not automatic. A fast contender may repeatedly reacquire while others starve. Use queued acquisition or randomized backoff when fairness matters.

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Holder pauses beyond lease | Two processes execute | Fencing at every protected resource |
| Network partition isolates holder | Old owner keeps acting | Higher-term takeover plus fencing |
| Old owner releases new lease | Successor loses ownership | Conditional release using lease ID/version |
| Renewal succeeds but reply is lost | Ownership is uncertain | Stop new work, re-read lease, enforce token |
| Lease duration is too short | Churn and duplicate attempts | Budget for pauses and store latency |
| Lease duration is too long | Slow failover | Tune from recovery objective and workload |
| Client clocks disagree | Premature/late expiry | Store-authoritative expiry; monotonic local timers |
| Random ID used as fence | Resource cannot order owners | Monotonic sequence or revision |
| Fence checked before, not with, write | Race between check and mutation | Atomic compare-and-write |
| One global lock | Throughput collapse | Scope locks to independent invariants |
| Multiple locks acquired inconsistently | Deadlock | Total ordering and bounded acquisition |
| Lock held over vendor call | Long contention; uncertain result | Durable command state and reconciliation |
| Coordination quorum lost | No new leader | Fail safe, preserve reads, use runbook |
| Watch event is missed | Stale local leadership view | Re-read after reconnect |
| Leader performs all work | Failover loses progress | Leader assigns durable idempotent tasks |
| External API lacks fencing/idempotency | Stale duplicate side effect | Controlled gateway, ledger, reconciliation, explicit limitation |

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Optimistic CAS | No lock while uncontended | Retries under contention |
| Pessimistic lock | Simple exclusive section | Blocking and deadlocks |
| Short lease | Fast recovery | False takeover during pauses |
| Long lease | Fewer false takeovers | Slow recovery |
| Central coordinator | Clear ordering | Quorum/availability dependency |
| Database coordination | Fewer components | Load and lifecycle coupling |
| Fine-grained locks | Parallelism | More keys and protocol complexity |
| Coarse lock | Simpler reasoning | Contention and blast radius |
| Fencing every write | Strong stale-owner defense | Resource integration work |
| Best-effort external serialization | Works with limited vendors | Cannot prove exactly-once behavior |
| Single global leader | Easy control | Bottleneck and large failure domain |
| Sharded leaders | Scale and isolation | Rebalancing complexity |

## Construction operations example

A platform closes monthly project costs. It reads approved invoices, computes committed cost, publishes a snapshot, and initiates an ERP close command.

Invariant: for each tenant, project, and accounting period, only the newest close attempt may publish authoritative snapshot state; an ERP close has one stable operation ID and must be reconciled if its response is unknown.

Design:

1. A scheduler leader creates durable close tasks; it does not perform every close itself.
2. Each worker acquires `tenant/project/period` with lease ID and token.
3. The database transitions the close operation using compare-and-swap.
4. Snapshot publication atomically accepts only a token greater than its stored token.
5. The ERP adapter accepts commands only from the current operation state and uses `close_operation_id` as the idempotency key.
6. If the ERP times out, the state becomes `outcome_unknown`; the worker does not blindly retry.
7. Reconciliation receives protected rate-limit capacity and queries the ERP by operation/reference ID.
8. Loss of the lease stops new work. A successor gets a higher token and resumes from durable state.
9. An old worker that resumes can neither overwrite the snapshot nor change the operation state because token/version checks fail.
10. If the ERP lacks idempotency or lookup, the design exposes the residual duplicate risk and routes ambiguity to an operator.

This combines coordination, fencing, durable workflow state, idempotency, and reconciliation. None replaces the others.

## Healthcare operations variation

A regional operations platform elects one coordinator per facility to build a bed-capacity view and assign refresh partitions. A coordinator lease does not grant clinical authority. Every read and proposed action still uses patient/facility authorization and current policy.

Derived census snapshots carry the coordinator term and source-data watermark. Storage rejects lower terms. If leadership changes during a pause, the old coordinator cannot publish. Durable tasks allow the new coordinator to resume.

Consequential clinical actions are not protected merely by a scheduler lock. They use authoritative clinical-system versions, explicit human authorization where required, idempotent operation IDs, and reconciliation. During quorum loss, the platform may serve clearly labeled last-known operational data within policy, but it does not invent a new leader or bypass the approved manual workflow.

## What to measure

Track acquisition latency and failures; lease renewal latency and uncertainty; ownership changes; lease age; fencing-token rejections; conditional-write conflicts; leader term and churn; time without a leader; quorum health; contention and wait time by key; deadlock/timeouts; task duplicates; stale-worker attempts; outcome-unknown external calls; reconciliation age; failover recovery time; and manual interventions.

Alert on repeated leadership churn, renewal latency approaching the safety margin, any accepted lower token, and singleton work that has made no durable progress.

## Interview prompt

> Design coordination for a multi-tenant operations platform. Exactly one scheduler per region should assign work, only one worker may publish each project/month cost snapshot, workers can pause for 90 seconds, and an ERP close call can time out after succeeding. The coordination store can lose quorum, and the ERP does not support fencing tokens.

Cover when to avoid locks, lease acquisition and renewal, fencing, leader election, external side effects, failure behavior, granularity, observability, and healthcare adaptation.

## Worked answer

**One-line conclusion:** Use consensus- or transaction-backed leases for liveness, monotonically increasing fencing tokens for safety, and durable idempotent workflows plus reconciliation for effects that a lock cannot make exactly once.

“I would first encode uniqueness and state transitions in the database, reserving leases for scheduler leadership and genuinely exclusive snapshot publication. The regional scheduler holds a renewable lease with a monotonically increasing term and only assigns durable idempotent tasks.

A worker acquires a project/month lease with a unique lease ID and token. Snapshot storage atomically accepts only tokens greater than the stored token. Renewal and release are conditional. If renewal is uncertain, the worker stops starting work; a new owner may take over after expiry. A 90-second pause can therefore create overlapping execution, but the old token cannot publish.

ERP commands flow through an operation ledger and controlled adapter. Each close has a stable operation ID. A timeout produces `outcome_unknown`, not an immediate duplicate call; reconciliation checks the ERP before retrying. Because the ERP cannot enforce fencing, I would not claim exactly-once execution. I would use any vendor idempotency or version feature, serialize commands at the adapter, and route unresolved ambiguity to an operator.

During coordination-store quorum loss, existing safe reads may continue, but no candidate invents leadership and consequential new work waits. I would monitor renewals, leader churn, fencing rejections, lock contention, durable progress, uncertain ERP outcomes, and recovery time.”

## Practice lab

Worker A holds token 71 for a project close, then pauses for 90 seconds. Its lease expires. Worker B acquires token 72, publishes the snapshot, and calls the ERP. The ERP succeeds but its response is lost. A resumes with valid network credentials while the coordination store is unavailable.

Specify:

- which actions A and B may still attempt;
- where tokens and versions are checked atomically;
- the lease/renewal timing policy;
- durable operation states;
- how the ERP adapter prevents or detects duplicates;
- behavior if the ERP has neither idempotency nor lookup;
- failover and quorum-loss behavior;
- metrics and an operator runbook.

Then adapt the design to a hospital bed-capacity coordinator without treating leadership as clinical authorization.

## References

- [The Chubby lock service for loosely-coupled distributed systems](https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/)
- [Apache ZooKeeper recipes and solutions](https://zookeeper.apache.org/doc/current/recipes.html)
- [etcd concurrency API](https://pkg.go.dev/go.etcd.io/etcd/client/v3/concurrency)
- [etcd API guarantees](https://etcd.io/docs/v3.5/learning/api_guarantees/)
- [PostgreSQL advisory-lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)
- [Martin Kleppmann — How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
