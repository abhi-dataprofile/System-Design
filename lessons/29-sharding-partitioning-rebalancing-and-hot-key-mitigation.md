# 29 — Sharding, partitioning, rebalancing, and hot-key mitigation

Previous: [Distributed locks, leases, fencing tokens, and leader election](28-distributed-locks-leases-fencing-tokens-and-leader-election.md). Study time: 55–65 minutes including the exercise.

## Mental model

Partitioning divides a large workload into independently manageable pieces. Sharding places those pieces across machines or administrative boundaries.

> A good partition key spreads load, keeps common operations local, preserves isolation, and can be moved without losing ownership or correctness.

A shard is not merely a storage bucket. It is a failure, scaling, routing, recovery, and often compliance boundary. Every design must answer four questions:

1. **Placement:** which shard owns a key now?
2. **Routing:** how does a request find that owner?
3. **Movement:** how does ownership change safely?
4. **Skew:** what happens when one key is much hotter or larger than others?

A simple hash can distribute keys evenly but scatter related records. A tenant key keeps work local but creates a hot shard when one tenant dominates. A time key makes retention and scans convenient but sends every current write to the newest partition. The partition key is therefore a workload decision, not a database checkbox.

## When to partition

Partition when one node, table, index, queue, or worker group can no longer satisfy a concrete limit:

- storage or index size;
- write or read throughput;
- connection, CPU, memory, or I/O pressure;
- maintenance, backup, or restore duration;
- tenant isolation or data residency;
- queue parallelism;
- search-index or vector-index size;
- blast-radius requirements;
- retention and deletion by time range.

Do not shard only because the system may grow someday. Sharding adds routing, cross-shard query, migration, operational, and consistency costs. First use sound indexes, caching, read replicas, batching, archival, backpressure, and vertical scaling when they meet the requirement.

Before sharding, quantify:

- current and projected records, bytes, requests, and work units;
- key cardinality and access distribution;
- largest tenant, project, facility, document, and time bucket;
- common read/write transactions;
- cross-key joins and aggregations;
- growth and retention;
- failure-domain and residency needs;
- acceptable migration load and cutover risk.

## Logical partitions versus physical shards

Separate the logical key space from current physical placement.

```text
request key
   → logical partition
      → placement-map version
         → physical shard and replica set
```

Use more logical partitions than physical shards. A new machine can then receive existing partitions rather than forcing every record to be rehashed. The placement map records ownership, state, replicas, and migration epoch.

Applications should not embed a permanent rule such as `hash(tenant_id) % 8`. Changing from eight to nine shards would move most keys and make old clients route incorrectly. Route through a versioned shard map, a proxy, or a service that can refresh placement metadata.

## Partition-key strategies

### Hash partitioning

`partition = hash(key) mod N`, or preferably hash to a larger fixed logical space.

Benefits:

- good average distribution;
- simple point lookup;
- parallel writes across unrelated keys.

Costs:

- range scans fan out;
- related records may be separated;
- naive modulo hashing causes large movement when N changes;
- even hashes cannot fix a single hot key.

Use for high-cardinality keys with mostly point access.

### Range partitioning

Assign ordered intervals such as date, project number, or customer ID ranges.

Benefits:

- efficient range scans;
- natural archival and retention;
- easy ordered traversal.

Costs:

- sequential keys create a hot tail;
- uneven ranges grow at different rates;
- split/merge logic is required.

Use when bounded range queries dominate and write distribution is understood.

### Tenant partitioning

Keep a tenant's operational data together.

Benefits:

- local transactions and tenant exports/deletion;
- clear isolation and residency;
- easier tenant-level recovery.

Costs:

- tenants differ dramatically in size and traffic;
- one large tenant can exceed a shard;
- fleet utilization may be uneven.

Use virtual buckets within a tenant or dedicate large tenants to their own placement class.

### Time partitioning

Group data by day, month, or retention period.

Benefits:

- cheap expiry, archival, and historical scans;
- immutable older partitions can use cheaper storage.

Costs:

- current-period hot spot;
- queries across long periods fan out;
- late events require old-partition writes or correction paths.

Often combine time with another dimension: `month + hash(tenant/project)`.

### Geographic or regulatory partitioning

Place data by approved region or jurisdiction.

Benefits:

- residency and latency control;
- regional fault isolation.

Costs:

- global views and tenant moves are harder;
- region loss and control-plane routing require explicit behavior.

Residency is a hard placement constraint, not a balancing hint.

### Composite partitioning

Combine dimensions, for example:

```text
region / tenant_bucket / time_bucket
```

Composite keys can balance locality and load, but every extra dimension complicates routing, rebalancing, and query planning.

## Choose from the access patterns

Create a table before choosing the key:

| Operation | Frequency | Required locality |
| --- | ---: | --- |
| Fetch invoice by ID | High | One partition |
| List invoices for project/month | High | Prefer one partition |
| Approve invoice and update workflow | Medium | Same transactional owner |
| Search across tenant documents | High | Fan-out may be acceptable only within tenant |
| Fleet-wide analytics | Low/async | Batch fan-out |
| Delete/export tenant data | Rare but mandatory | Discover all tenant partitions |
| Reconcile ERP outcomes | High priority | Stable operation-key routing |

Co-locate records that must transact atomically. Denormalize carefully for read locality, with versioned events and reconciliation for derived copies.

A global secondary index can restore an access path that the primary partition key breaks, but the index is itself a distributed system. Define update atomicity, staleness, backfill, deletion, and failure semantics.

## Consistent hashing and rendezvous hashing

Consistent hashing maps nodes and keys onto a ring. Adding or removing a node moves only adjacent ranges instead of most keys. Virtual nodes improve average balance and let heterogeneous machines own different shares.

Rendezvous hashing scores each key against candidate nodes and chooses the highest score. It offers minimal movement and can be simpler to implement for a moderate node set.

Neither algorithm solves:

- a single extremely hot key;
- large differences in record size;
- correlated tenant/time traffic;
- cross-shard transactions;
- residency constraints;
- safe data copying and cutover.

Hash balance is not work balance. Measure bytes, requests, CPU, tokens, queue age, and downstream calls—not only key count.

## Directory-based routing

A directory explicitly maps logical partitions or tenants to shards.

Benefits:

- arbitrary placement;
- dedicated shards for large tenants;
- residency and hardware constraints;
- controlled migration and rollback.

Costs:

- metadata service availability;
- cache invalidation;
- stale routes;
- placement-map operational complexity.

Routers cache the map with a version and bounded lifetime. A shard receiving a request for a moved partition should return a typed redirect or ownership error containing no sensitive data. Clients refresh and retry only if the operation is idempotent or carries a stable operation ID.

The control plane changes placement. The data plane should continue serving with the last valid map during a short control-plane outage when safe.

## Rebalancing protocol

Moving a partition while reads and writes continue is a state-machine problem.

Example states:

```text
SOURCE_OWNED
  → COPYING
  → CATCHING_UP
  → CUTOVER_PENDING
  → DESTINATION_OWNED
  → SOURCE_RETIRED
```

A robust online move:

1. choose a partition using measured load, capacity, constraints, and migration budget;
2. assign a migration ID and placement epoch;
3. create the destination replica without making it authoritative;
4. copy a consistent snapshot from a known source watermark;
5. capture concurrent changes through a log, CDC stream, or dual-write journal;
6. replay changes idempotently until lag is within the cutover threshold;
7. briefly fence or version-check writes while ownership changes;
8. atomically publish the new placement epoch;
9. route new reads and writes to the destination;
10. keep the source in read-only shadow mode during a rollback window;
11. compare counts, checksums, sampled records, and business invariants;
12. retire the old copy only after verification and retention policy permit it.

Blind application-level dual writes are dangerous: one write may succeed and the other fail. Prefer a single authoritative write plus replicated log. If dual writes are unavoidable, persist an outbox and reconcile discrepancies.

Every write should carry the partition ID and expected epoch. The destination rejects writes from an older epoch; this is fencing applied to shard ownership.

## Read behavior during movement

Choose explicitly:

- **source reads until cutover:** simplest, destination warms in shadow;
- **destination reads after watermark:** faster validation but requires version comparison;
- **read both and compare:** useful for migration verification, expensive;
- **proxy/forward from old owner:** smooths stale routers, adds a hop and must be bounded.

Do not serve whichever copy responds first if versions can diverge. Define authoritative ownership and read-your-write behavior.

Long-lived requests and queued jobs may contain an old shard address. Store logical keys, not only physical destinations, so a worker can reroute using the current epoch.

## Hot partitions and hot keys

A hot partition has excessive aggregate load. A hot key is one logical item whose traffic cannot be spread by ordinary hashing.

Diagnose before fixing:

- request rate and work units per partition/key;
- read versus write concentration;
- bytes, CPU, I/O, connection, and lock pressure;
- cache hit rate;
- queue age and tail latency;
- tenant and operation mix;
- time-of-day pattern;
- downstream dependency use.

### Read-hot mitigation

- cache immutable or versioned results;
- replicate the hot item and route reads across replicas;
- request coalescing/single-flight on cache miss;
- precompute materialized views;
- CDN or edge cache for public artifacts;
- stale-while-revalidate within a defined freshness budget;
- admission control for expensive refreshes.

A cache key must include tenant, authorization-relevant scope, version, and representation. Do not turn load mitigation into a data leak.

### Write-hot mitigation

Writes to one counter or record require more care:

- batch increments locally and merge;
- use sharded counters with an asynchronous aggregate;
- append events to multiple buckets, then reduce;
- serialize only the narrow invariant;
- use compare-and-swap with bounded retry;
- separate one overloaded tenant into subpartitions;
- redesign the invariant to be commutative where possible.

Sharded counters trade exact instantaneous reads for throughput. If an invoice approval must be exact, do not replace it with an eventually consistent counter merely to scale.

### Key salting

Append a bucket number to spread work:

```text
tenant_42/project_91/bucket_0..31
```

Writers choose a bucket; readers fan out and merge. Salting works for associative data such as counts, events, or document chunks. It increases read cost and is unsuitable when all writes must enforce one record-level invariant.

### Split a large tenant

Use a stable second dimension such as project, facility, document, or hash bucket. Maintain a tenant directory that lists its subpartitions.

Do not split solely by a random field if common tenant/project transactions then require expensive fan-out.

## Cross-shard operations

Avoid synchronous distributed transactions on the hottest path when a local invariant plus durable workflow is sufficient.

Patterns:

- choose one shard as authoritative for the workflow;
- store a durable saga/operation record;
- emit an outbox event;
- apply idempotent updates to other shards;
- expose pending state rather than false success;
- reconcile until all required projections converge;
- compensate only when a real business inverse exists.

For cross-shard queries:

- scatter/gather with a strict partition and deadline bound;
- maintain a purpose-built global index;
- stream changes into an analytical store;
- precompute tenant-level summaries;
- paginate with per-shard cursors and a stable ordering contract.

Never let an unbounded fleet-wide fan-out sit behind an interactive API.

## Multi-tenant placement

Classify tenants by observed needs:

- pooled shards for small tenants;
- bucketed placement for medium tenants;
- dedicated shard or cluster for exceptionally large, regulated, or noisy tenants.

Promotion from pooled to dedicated is a migration, not a manual database copy. Preserve logical identifiers, authorization, encryption ownership, audit continuity, backups, deletion coverage, and rollback.

Fairness is still required inside a shard. Balanced storage does not prevent one tenant's backfill from consuming every connection or model token.

Maintain a reverse inventory: given a tenant, list every primary shard, replica, search partition, object prefix, cache namespace, derived dataset, and backup policy. This is essential for export, deletion, incident response, and residency evidence.

## AI and retrieval systems

AI workloads partition several different resources:

- source documents in object storage;
- metadata in an operational database;
- text/vector search indexes;
- ingestion queues;
- embedding and model capacity;
- evaluation datasets and traces.

Use the same stable logical document and tenant IDs across layers, but do not assume every layer needs the same physical partition key.

Vector/search partitioning choices affect recall and latency. Tenant filters must be enforced before returning candidates, not merely applied after retrieving cross-tenant results. A very large tenant may require multiple vector partitions plus a tenant-scoped router. Global fan-out should be bounded, and migration must preserve embedding-model and source-version lineage.

When rebalancing an index:

1. build the destination from a source watermark;
2. replay changes;
3. query both in shadow;
4. compare recall, authorization filtering, and latency;
5. atomically switch the index alias or routing epoch;
6. retain rollback until verification completes.

## Construction operations example

A construction platform serves 300 contractors. One national GC generates 35% of invoice and document traffic, while a single active project creates a month-end spike. Users query project costs, search drawings, approve invoices, and reconcile ERP writes.

Design:

1. use region as a hard residency boundary;
2. route small tenants to pooled logical partitions;
3. partition the large GC by `tenant + project_bucket`;
4. co-locate each invoice, approval state, and operation ledger by project;
5. keep tenant/project/month summaries as versioned derived views;
6. salt append-only drawing chunks within a large project, but keep document metadata authoritative in one project partition;
7. dedicate queue lanes and quotas so its backfill cannot starve other tenants;
8. use a directory-based map with placement epochs;
9. move a hot project through snapshot, CDC catch-up, fenced cutover, shadow comparison, and rollback window;
10. keep ERP operation IDs logically routed so reconciliation follows a moved project;
11. send portfolio analytics to an analytical store instead of synchronously scanning operational shards;
12. inventory every partition holding the tenant for export, retention, and deletion.

If the large project's invoice-approval row becomes write-hot, do not randomly salt it. Preserve one authoritative state machine and scale surrounding reads, evidence, and events separately.

## Healthcare operations variation

A healthcare operations platform partitions first by approved region and organization, then by facility or patient hash according to transaction patterns. PHI never crosses residency boundaries merely to improve balance.

A large hospital can receive dedicated placement while smaller clinics share pooled shards. Facility census summaries are derived and versioned; patient-level authoritative updates stay with the appropriate clinical-system boundary. Cross-facility operational analytics run from governed replicas or an analytical store.

During migration, both source and destination preserve encryption, audit, access policy, retention, and break-glass controls. The placement epoch fences stale writes. Shadow reads compare correctness without exposing PHI in migration logs. If routing metadata is unavailable, the platform serves only operations safely routable from the last valid map and does not guess a region or tenant.

Partitioning improves capacity and blast radius; it does not weaken clinical authorization or make eventually consistent data suitable for consequential clinical decisions.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Hash partitioning | Even average key distribution | Poor range locality |
| Range partitioning | Efficient scans and retention | Hot tail and uneven growth |
| Tenant partitioning | Isolation and local operations | Large-tenant skew |
| Time partitioning | Easy archival | Current-period hot spot |
| Directory routing | Flexible placement | Metadata dependency |
| Consistent hashing | Limited key movement | Weak fit for hard constraints |
| Many logical partitions | Fine rebalancing | More metadata and overhead |
| Dedicated large-tenant shard | Isolation and predictability | Lower pooled utilization |
| Online migration | Little downtime | Complex correctness protocol |
| Stop-the-world move | Simpler consistency | User-visible unavailability |
| Salting | Spreads associative work | Fan-out reads and merge cost |
| Global secondary index | Alternate lookup path | Consistency and backfill complexity |
| Cross-shard transaction | Strong atomicity | Latency and availability coupling |
| Saga/derived projection | Resilient local progress | Temporary inconsistency |
| Read replication | Hot-read relief | Freshness and failover semantics |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Key count looks balanced but work is not | One shard overloads | Weight by bytes and resource use |
| Tenant ID is the only key | Large tenant dominates | Subpartition or dedicate it |
| Timestamp is leading key | All new writes hit one shard | Add hash/bucket dimension |
| Naive modulo changes | Most records move | Stable logical partitions |
| Router caches stale ownership | Writes hit old shard | Epoch checks, typed redirect, refresh |
| Source and destination both accept writes | Divergent histories | Single authority and fencing |
| Dual write partially fails | Missing destination data | Log/outbox and reconciliation |
| Copy misses concurrent changes | Silent data loss | Snapshot watermark plus CDC |
| Cutover occurs before catch-up | Stale reads or lost writes | Lag threshold and validation gate |
| Old jobs store physical address | Work executes on wrong shard | Resolve logical key at execution |
| Hot key is moved repeatedly | Load follows it | Cache, split semantics, or redesign |
| Salting exact state | Broken invariant | Keep exact state authoritative |
| Fan-out is unbounded | Tail-latency collapse | Bound partitions/deadlines or precompute |
| Global index is stale | Wrong routing/result | Version and verify authoritative record |
| Rebalance saturates production | Incident caused by migration | Rate limit and pause automatically |
| Tenant migration misses derived copy | Incomplete deletion/export | Reverse inventory and reconciliation |
| Region chosen only for load | Residency breach | Enforce placement constraints |
| Search repartition drops filters | Cross-tenant exposure | Mandatory pre-result authorization |

## What to measure

Track storage bytes, records, requests, weighted work, CPU, memory, I/O, connections, cache misses, queue age, and tail latency per logical partition, shard, tenant, and operation. Measure skew ratios and top-key share.

For migration track copied bytes, snapshot watermark, CDC lag, retries, checksum mismatches, shadow-read differences, stale-route redirects, rejected old epochs, cutover duration, rollback readiness, and production headroom consumed.

For cross-shard work track fan-out width, partial results, deadline exhaustion, saga age, reconciliation backlog, and global-index lag. Alert on sustained skew, a partition approaching hard capacity, migration lag growing, or any accepted write from an old epoch.

## Interview prompt

> Design data and work partitioning for a multi-tenant construction AI platform. It serves 300 tenants, one tenant generates 35% of traffic, one project becomes hot at month-end, and data must stay in its approved region. The system stores invoices, approval workflows, documents, vectors, queues, and ERP operation records. Add capacity without downtime and preserve tenant isolation during migration.

Cover partition keys, routing, hot-key mitigation, online rebalancing, ownership fencing, cross-shard queries and workflows, AI indexes, failure handling, observability, and healthcare adaptation.

## Worked answer

**One-line conclusion:** Partition by stable business locality under hard residency constraints, route through versioned logical ownership, and rebalance with snapshot-plus-log catch-up and fenced cutover while treating hot keys as workload-design problems rather than hashing problems.

“I would start from access patterns and invariants. Region is a hard boundary. Small tenants use pooled logical partitions; the dominant tenant is split by project bucket, keeping each invoice, approval workflow, and ERP operation ledger co-located. Portfolio analytics and broad historical queries go to a derived analytical store rather than fan out across the transactional fleet.

A versioned directory maps logical partitions to physical shard replica sets. Requests carry the expected placement epoch, and shards reject stale ownership. Routers refresh on typed redirects. Queued jobs retain logical tenant/project keys so they can reroute after a move.

For online rebalancing, I create a destination, copy a snapshot at a known watermark, replay CDC idempotently, shadow reads, and verify counts, checksums, tenant isolation, and business totals. At low lag I fence writes, atomically publish the new epoch, send traffic to the destination, and hold the source read-only through a rollback window. Migration traffic is rate-limited and pauses if production headroom falls.

For the hot project, I cache versioned evidence and summaries, coalesce misses, and isolate its queues and quotas. Append-only document chunks may be salted and merged, but the exact invoice approval state remains on one authoritative partition. If that invariant itself exceeds one node, I redesign the workflow boundary rather than salt correctness away.

Search uses tenant-scoped routing and enforces authorization before results. Index moves use source-version watermarks, shadow recall tests, and atomic aliases. I monitor per-partition weighted load, skew, hot-key share, stale routes, migration lag, checksum differences, old-epoch rejections, fan-out width, and reconciliation age.”

## Practice lab

Design the partitioning plan when:

- 300 tenants share four shards;
- one tenant owns 35% of traffic and 28% of data;
- one active project produces 60% of that tenant's writes at month-end;
- invoice approval must be strongly consistent;
- document search and portfolio analytics span projects;
- an index and database partition must move online;
- workers may hold stale routes for ten minutes;
- ERP reconciliation tasks are already queued;
- a destination fails midway through catch-up;
- the tenant requests export and deletion during migration.

Specify the partition key, logical partition count, placement metadata, hot-read and hot-write controls, snapshot/CDC watermark, ownership epoch, cutover and rollback gates, cross-shard query plan, queue rerouting, deletion inventory, and metrics.

Then adapt the plan to a hospital group with facility-level residency and patient-level authorization.

## References

- [Amazon Dynamo: a highly available key-value store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Google Cloud Spanner — Schema design best practices](https://cloud.google.com/spanner/docs/schema-design)
- [Apache Cassandra — Data modeling](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/intro.html)
- [MongoDB — Sharding](https://www.mongodb.com/docs/manual/sharding/)
- [PostgreSQL — Table partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [Azure Architecture Center — Data partitioning guidance](https://learn.microsoft.com/azure/architecture/best-practices/data-partitioning)
