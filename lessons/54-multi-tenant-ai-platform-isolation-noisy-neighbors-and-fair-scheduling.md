# Multi-tenant AI platform isolation, noisy neighbors, and fair scheduling

A multi-tenant AI platform shares expensive infrastructure without allowing one tenant’s data, identity, failures, or demand to affect another tenant incorrectly.

Ordinary SaaS isolation is necessary but insufficient. AI platforms add prompts, retrieved chunks, embeddings, adapters, KV caches, batch schedulers, human-review queues, tool credentials, and model-serving memory. Each can become a cross-tenant leak or noisy-neighbor path.

## Mental model: isolation is a chain of enforced boundaries

Treat every request as a tenant-scoped capability moving through shared stages:

```text
identity
 -> policy
 -> data and retrieval
 -> queue and scheduler
 -> model runtime and cache
 -> tools
 -> telemetry and outcome
```

At every stage answer:

1. **Who is the tenant and principal?**
2. **Which resource and action are authorized?**
3. **What state can be shared safely?**
4. **What resource budget is consumed?**
5. **What failure or timing signal can cross the boundary?**

Isolation has several dimensions:

- **confidentiality:** tenant A cannot observe tenant B’s data or derived state;
- **integrity:** tenant A cannot change tenant B’s workflows, models, or policy;
- **availability:** tenant A cannot exhaust shared capacity;
- **performance:** latency remains bounded under other tenants’ bursts;
- **economic:** usage and spend are attributed and limited correctly;
- **operational:** incidents, backups, logs, and support access preserve boundaries.

A tenant ID supplied by the model or client is data, not authority. Derive scope from authenticated context and enforce it at every resource server.

## When to share and when to dedicate

Shared infrastructure is appropriate when workloads are numerous, bursty, similarly regulated, and small enough to benefit from statistical multiplexing.

Use dedicated pools, accounts, indexes, encryption domains, or deployments when tenants require:

- contractual physical or cryptographic isolation;
- separate residency or regulatory controls;
- custom models, adapters, or network connectivity;
- predictable high sustained demand;
- a smaller blast radius;
- independent release schedules or support access.

Isolation is a spectrum. A common platform may offer shared, pooled-with-reservations, dedicated-compute, and fully isolated tiers. Make the guarantees measurable and explicit.

## Tenant context and authorization

Create tenant context once at a trusted ingress after authentication. Include subject, tenant, role, purpose, region, workload, and request ID. Sign or protect that context across service boundaries.

Every data, cache, queue, model, and tool operation must require server-side tenant scope. Prefer APIs where tenant scope is structurally unavoidable:

- tenant-specific database connection or row-security session;
- namespaced object paths and encryption keys;
- tenant-bound queue topics and consumer claims;
- retrieval filters injected by the service, not the model;
- tool credentials scoped to tenant resources;
- cache keys that include tenant, authorization, model, policy, and source versions.

Do not accept a tenant identifier embedded in a prompt, document, tool argument, or model output as authorization.

## Data and retrieval isolation

Common storage patterns include:

| Pattern | Isolation | Operational tradeoff |
|---|---|---|
| Shared tables with tenant key | Efficient and simple to operate | Every query and migration must preserve scope |
| Schema per tenant | Stronger logical separation | Migration and connection overhead |
| Database or account per tenant | Strong blast-radius and key boundaries | Higher cost and fleet complexity |
| Dedicated deployment | Maximum customization and control | Lowest pooling efficiency |

Defense in depth matters. Combine application checks with database row policies, tenant-bound keys, object-store policies, and automated negative tests.

For vector retrieval:

- store immutable tenant and resource ACL metadata with each chunk;
- apply authorization filtering before or during retrieval;
- verify returned records after retrieval;
- separate indexes when filters cannot provide adequate guarantees;
- prevent global deduplication from exposing existence or content;
- include tenant and ACL versions in embedding and cache lineage;
- test deletion, reindexing, backup, and restore paths.

A globally nearest vector is irrelevant if it is unauthorized.

## Model, adapter, and cache isolation

Shared inference introduces state that may outlive a request:

- KV and prefix caches;
- continuous-batching buffers;
- fine-tuned adapters;
- tokenizer and preprocessing buffers;
- temporary files and GPU memory;
- speculative decoding state;
- output and safety caches.

Tag every allocation with tenant and request ownership. Zero or securely recycle memory before reassignment when the runtime does not guarantee isolation. Never reuse semantic or prefix caches across tenants unless the cached input is explicitly public, content-addressed, policy-compatible, and incapable of revealing access patterns.

Load adapters from signed tenant-scoped manifests. Verify that a request cannot name another tenant’s adapter. Bound the number and size of resident adapters so one tenant cannot evict every other tenant’s working set.

Side channels matter: cache-hit latency, error differences, queue position, token streaming, and rate-limit headers can reveal another tenant’s activity. Eliminate high-risk signals and document residual leakage.

## Noisy neighbors

A tenant can dominate:

- request concurrency;
- input and output tokens;
- GPU memory and decode slots;
- OCR pages or image pixels;
- embedding and index writes;
- storage and network;
- tool calls;
- human-review capacity;
- control-plane operations such as adapter or index loading.

Measure work in resource units, not only requests. Estimate cost at admission and settle actual cost after completion. Use separate limits for sustained rate, bursts, concurrency, queued work, storage, and spend.

Protect control traffic, revocations, health checks, and safety-critical work from data-plane saturation.

## Fair scheduling

Fairness does not mean equal request counts. Tenants have different contracts, workload sizes, and priorities.

Use hierarchical scheduling:

```text
platform reserve
  -> service class
     -> tenant share
        -> workload queue
           -> request
```

Useful mechanisms:

- **weighted fair queuing:** allocate service proportional to contractual weights;
- **deficit round robin:** serve variable-sized jobs using cost estimates and credits;
- **dominant-resource fairness:** account for the resource each workload stresses most;
- **reservations plus borrowing:** guarantee a minimum while lending unused capacity;
- **priority with aging:** serve urgent work while preventing indefinite starvation;
- **deadline-aware admission:** reject or defer work that cannot meet its SLO.

Cap borrowing and reclaim it gradually. A tenant using idle capacity should not acquire a permanent entitlement. Avoid strict priority without bounds; low-priority indexing may otherwise starve forever.

Maintain separate interactive and batch queues. Large prompts and long generations should not sit ahead of short interactive work simply because they arrived first.

## Tenant placement and blast radius

Place tenants using constraints such as region, data class, encryption domain, dedicated tier, model eligibility, and capacity. Only then optimize load.

Use cells or stamps: small, independently deployable groups of data stores, queues, caches, and serving pools. A global control plane maps tenants to cells; the data plane operates locally.

Cell benefits include bounded incidents, staged releases, simpler capacity reasoning, and region isolation. Costs include fragmentation, uneven utilization, tenant movement, and more operations.

Moving a tenant requires an explicit state machine: copy versioned data, dual-read or validate, freeze writes or replicate deltas, switch authoritative routing, verify, and retire the old placement. Never update a routing table before state and keys are ready.

## Control-plane isolation

Tenant configuration, policy, quotas, model eligibility, keys, and placement are high-value control-plane state.

- use optimistic concurrency or versioned updates;
- audit every change with actor and reason;
- validate cross-field invariants before publication;
- distribute immutable snapshots;
- make data planes continue with a bounded last-known-good policy;
- prevent one tenant’s configuration storm from exhausting global control capacity;
- separate support impersonation from ordinary operator access and require approval.

A control-plane outage should not silently remove tenant filters or quotas.

## Observability and support access

Metrics need tenant dimensions for attribution and fairness, but raw tenant IDs in high-cardinality metric labels can overload monitoring. Use aggregated metrics, exemplars, traces, and an access-controlled usage ledger.

Logs and traces must not mix prompts or retrieved content across tenant support views. Redact sensitive payloads and bind diagnostic access to tenant, purpose, case, and expiration.

Support tooling is part of the isolation boundary. “Super admin” should not imply unrestricted content access. Prefer metadata-first diagnosis, audited break-glass access, and tenant-visible support events where appropriate.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Shared serving pool | High utilization and elastic borrowing | Larger side-channel and noisy-neighbor surface |
| Dedicated tenant pool | Predictable performance and blast radius | Stranded capacity and operational cost |
| Shared vector index with filters | Efficient operations | Filter bugs and difficult deletion verification |
| Separate tenant indexes | Clearer isolation and lifecycle | More shards, memory, and rebalancing |
| Work-conserving borrowing | Uses idle capacity | Reclaim latency and expectation management |
| Hard reservations | Strong guarantees | Capacity sits idle |
| Fine-grained tenant telemetry | Attribution and debugging | Cardinality, privacy, and cost |
| Cell architecture | Bounded failures and releases | Fragmentation and tenant-movement complexity |
| Global cache sharing | Better hit rate | Confidentiality and access-pattern risk |
| Tenant-scoped caches | Safer semantics | Lower hit rate and more memory |

## Failure modes and defenses

| Failure mode | Defense |
|---|---|
| Model supplies another tenant’s ID | Tenant context comes only from authenticated ingress and resource authorization |
| Retrieval filter is omitted | Mandatory server-side policy, post-filter verification, and cross-tenant negative tests |
| Cache key lacks tenant or policy version | Typed cache-key builder and tenant-scoped encryption |
| GPU worker exposes prior request memory | Runtime isolation validation, memory clearing, process boundaries, or dedicated pools |
| One long prompt blocks interactive traffic | Cost-aware admission, separate queues, preemption or bounded scheduling quanta |
| Premium tenant borrows all capacity | Reservations, borrowing caps, reclaim policy, and protected reserve |
| Low-priority batch work never runs | Aging, bounded priority, and batch capacity windows |
| One tenant’s index rebuild saturates storage | Per-tenant I/O budgets and isolated background queues |
| Shared adapter cache thrashes | Per-tenant residency limits and admission-aware loading |
| Control-plane outage disables policy | Signed last-known-good snapshots and fail-closed changes |
| Backup restore mixes tenant data | Tenant-aware manifests, restore validation, and isolated recovery tests |
| Support engineer sees unrestricted prompts | Case-bound, time-limited, audited break-glass access |
| Tenant move causes split ownership | Versioned placement state machine and single authoritative epoch |

## Construction example: shared project AI platform

A construction group hosts invoice extraction, drawing search, and RFI drafting for multiple owners, joint ventures, and subcontractors. Some project numbers overlap, and one hospital project has stricter residency and support-access rules.

Authenticated project membership produces a tenant-and-project capability. Retrieval scope is injected by the service and checked again on returned drawing chunks. Each tenant has separate encryption context, cache namespace, adapter manifest, and usage ledger.

A large drawing reindex from one owner floods embedding work. It enters the batch queue with a tenant I/O and accelerator budget, while interactive RFI questions use reserved capacity. Unused interactive capacity can be borrowed, but the scheduler reclaims it when latency rises.

The hospital project is placed in a dedicated cell with an eligible model pool and restricted diagnostics. A general pool failure cannot cause it to route to an unapproved region. When a joint venture moves to a dedicated tier, a versioned placement workflow copies data, catches up deltas, validates keys and indexes, switches one routing epoch, and then retires the old copy.

A healthcare-operations platform uses the same pattern for facilities or provider groups, with patient data requiring an additional authorization dimension. Tenant isolation never replaces patient-level access control.

## Interview prompt

**Design a multi-tenant AI platform for construction companies that shares model serving, OCR, vector search, tools, and human review. Large tenants can burst, some projects require dedicated regional isolation, and no tenant may leak data or starve others.**

## Worked answer

1. **Clarify guarantees.** Define tenant/project hierarchy, data classes, residency, workload sizes, SLOs, contract weights, dedicated tiers, and recovery objectives.
2. **Establish trusted context.** Authenticate at ingress and propagate protected subject, tenant, project, purpose, region, and request identity. Resource servers independently authorize it.
3. **Isolate data.** Choose shared, schema, database, or dedicated patterns by tier; add row policies, tenant keys, namespaced objects, tenant-aware backup, and negative tests.
4. **Secure retrieval and caches.** Apply mandatory ACL filters, verify results, include tenant and policy versions in cache keys, and prohibit unsafe cross-tenant semantic caching.
5. **Isolate inference state.** Tag buffers, KV caches, adapters, temporary files, and GPU allocations; clear or separate state and validate the runtime.
6. **Meter real work.** Estimate and settle tokens, memory, pages, I/O, tools, storage, and review—not request count alone.
7. **Schedule hierarchically.** Reserve platform and safety capacity, then allocate service-class and weighted tenant shares with bounded borrowing, aging, and separate batch queues.
8. **Bound blast radius.** Place tenants into regional cells after policy filtering; use versioned placement and safe migration state machines.
9. **Protect the control plane.** Publish validated immutable policy snapshots, audit changes, rate-limit configuration work, and retain bounded last-known-good behavior.
10. **Verify continuously.** Run cross-tenant access, cache, backup/restore, adapter, GPU-memory, queue-fairness, cell-failure, and support-access tests. Monitor isolation violations, SLOs, starvation, borrowing, and unallocated usage.

**One-line conclusion:** A trustworthy shared AI platform derives tenant authority outside the model, isolates every stateful layer, and schedules measured resource consumption with explicit guarantees and bounded borrowing.

## References

- [NIST SP 800-210 — General Access Control Guidance for Cloud Systems](https://csrc.nist.gov/pubs/sp/800/210/final)
- [NIST SP 800-207 — Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [Kubernetes Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [Kubernetes Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Weighted Fair Queueing: A Packetized Approximation of Generalized Processor Sharing](https://dl.acm.org/doi/10.1145/75247.75248)
- [Dominant Resource Fairness: Fair Allocation of Multiple Resource Types](https://www.usenix.org/conference/nsdi11/dominant-resource-fairness-fair-allocation-multiple-resource-types)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
