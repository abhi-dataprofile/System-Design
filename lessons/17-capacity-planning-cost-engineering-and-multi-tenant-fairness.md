# 17 — Capacity planning, cost engineering, and multi-tenant fairness

Previous: [Incident response, graceful degradation, and recovery](16-incident-response-graceful-degradation-and-recovery.md). Study time: 50–60 minutes including the exercise.

## Mental model

Capacity planning answers whether the system can meet its service objectives under expected load, bursts, failures, and growth. Cost engineering answers whether it can do so economically. Multi-tenant fairness decides whose work proceeds when demand exceeds supply.

Treat capacity as a flow problem:

```text
arrival rate → admission → queue → service → completion
```

For a stable queue, long-run service capacity must exceed admitted arrival rate. If average arrival rate is λ requests/second and average time in the system is W seconds, Little’s Law gives average concurrency:

```text
L = λ × W
```

This is a starting estimate, not a guarantee. Tail latency, burstiness, variable document size, retries, model routing, external quotas, failures, and cold starts determine production headroom.

> Plan from user-visible workload and bottleneck service demand, then allocate scarce capacity explicitly by tenant, priority, and risk.

A system running at 100% utilization has no room for bursts, failover, deployments, rebalancing, or slow requests. High utilization can improve unit economics while destroying latency and resilience.

## When to use each capacity technique

| Technique | Use it for | Limitation |
| --- | --- | --- |
| Back-of-envelope estimate | Early architecture and interview sizing | Hides distributions and bottlenecks |
| Historical percentile forecast | Known seasonal workloads | Misses product or tenant step changes |
| Load test | Validate latency, saturation, and failure behavior | Test realism and environment matter |
| Queueing model | Reason about utilization and waiting | Simplifying distribution assumptions |
| Shadow/replay | Candidate versions on realistic input | Privacy and duplicate load |
| Stress test | Find break point and degradation path | Must isolate side effects |
| Soak test | Leaks, drift, compaction, long workflows | Slow and costly |
| Failure/chaos test | Headroom during dependency loss | Requires bounded blast radius |
| Cost attribution | Optimize cost per successful outcome | Shared-cost allocation is approximate |

Use estimates to choose a design, tests to validate it, and production measurements to recalibrate it.

## Define workload units

Requests are not interchangeable. “1,000 AI requests” can mean short classification or long multimodal reasoning.

Define resource-oriented workload units:

- API request by endpoint and payload bucket;
- document page, byte, or extracted chunk;
- search query with retrieved candidates and rerank depth;
- input and output tokens by model;
- image/audio seconds or pixels;
- tool call and external API quota;
- workflow step and end-to-end completion;
- database rows scanned/written;
- object-storage bytes and requests;
- human-review minute.

For each class, measure arrival distribution, service demand, deadline, concurrency, retry amplification, cacheability, priority, and cost. Capacity plans should preserve the mix, not only total request count.

## Build a demand model

Separate:

- **baseline:** normal hourly/weekly pattern;
- **organic growth:** tenants, users, projects, documents;
- **events:** month-end invoice runs, shift changes, discharge rounds;
- **bursts:** one tenant bulk-imports 50,000 files;
- **recovery:** backlog drains after outage;
- **failover:** one region or provider carries displaced load;
- **deployment:** old/new fleets overlap;
- **adversarial/accidental:** loops, retry storms, or abusive input.

Example:

```text
peak interactive queries = 40 requests/s
mean end-to-end time = 2.5 s
Little's Law concurrency = 100
burst and tail multiplier = 2
one-zone failure factor = 1.5
planned concurrency ≈ 300
```

Do not multiply safety factors blindly. State what each covers, test the combined scenario, and avoid double-counting.

## Bottleneck and service demand

End-to-end capacity is bounded by the scarcest stage:

```text
admission → API → database/search → model → tool → external system
```

For stage i:

```text
required capacity_i =
  admitted rate × visits per request × service demand_i × headroom
```

Retries increase visits. A model timeout retried twice can triple provider demand while lowering successful throughput.

Measure saturation indicators appropriate to each resource:

- CPU run queue and throttling;
- memory pressure, allocation, and eviction;
- database connections, locks, IOPS, replication lag;
- cache hit rate and eviction;
- queue age and projected drain time;
- model concurrency, batching occupancy, KV-cache/GPU memory;
- provider token/request quota;
- external connector concurrency and rate-limit headers;
- human-review backlog and age.

CPU alone is not capacity. A worker may be blocked on model streams, database connections, GPU memory, or external quotas while CPU appears idle.

## Queueing and utilization

As utilization approaches the service limit, waiting time often grows nonlinearly. Therefore target utilization depends on variability and latency objectives.

Reserve headroom for:

- normal bursts and tail service times;
- instance/zone/provider failure;
- autoscaling delay and cold start;
- deployments and maintenance;
- retry/backfill/reconciliation work;
- forecast error.

Batch workloads can run at high utilization if deadlines tolerate queuing. Interactive or safety-critical workloads require lower targets and isolated capacity.

Use queue age, not only depth. Ten thousand 10 ms jobs differ from ten 20-minute jobs. Track service rate and estimate:

```text
drain time ≈ queued work / (service rate - new arrival rate)
```

If arrival rate is at or above service rate, the queue never drains without admission changes or more capacity.

## Autoscaling

Scale on the signal closest to scarce work:

| Workload | Better signal |
| --- | --- |
| Stateless API | concurrency, latency, CPU with request rate |
| Queue workers | queue age and work-weighted backlog |
| Streaming LLM | active sequences, tokens in flight, KV-cache pressure |
| Ingestion | pages/bytes awaiting parse |
| Database | often scale deliberately; protect with admission/queries |
| External connector | vendor quota and in-flight calls |

Autoscaling is delayed control. Provisioning, model loading, cache warming, and scheduling can take minutes. Use predictive or scheduled capacity for known peaks and warm pools for slow-start resources.

Set maximums to protect budgets and dependencies. Scale-in needs stabilization, connection draining, lease handling, and workflow checkpointing. Rapid scale oscillation wastes cost and interrupts work.

## AI inference capacity

LLM capacity depends on input length, output length, model, hardware/provider, batching, quantization, and latency target.

Separate:

- **prefill:** processes input tokens; compute-heavy and affected by context length;
- **decode:** generates tokens sequentially; memory-bandwidth and scheduling sensitive;
- **time to first token:** important for interactive experience;
- **inter-token latency:** streaming quality;
- **total completion time:** workflow deadline.

Continuous batching improves throughput by combining active requests, but large or long generations can delay small interactive requests. Use length buckets, deadlines, maximum tokens, preemption where supported, and separate pools for batch versus interactive work.

Context is not free. Retrieval that sends 10× more tokens can raise cost, reduce capacity, and sometimes reduce quality. Measure cost and outcome per successful task rather than cost per call.

For external model APIs, capacity also includes requests/minute, tokens/minute, regional availability, per-model quotas, and account limits. A fallback provider needs pre-negotiated quota and evaluated compatibility; it cannot absorb a regional outage if normally provisioned only for 5% traffic.

## Cost model

Build cost bottom-up:

```text
cost per successful workflow =
  (compute + model tokens + storage + database/search
   + network + observability + external APIs + human review
   + allocated idle/headroom + failed/retried work)
  / successful workflows
```

Track marginal cost and allocated shared cost. Idle failover capacity is not waste if it buys an explicit recovery objective.

Segment cost by tenant tier, workflow, model/prompt/index version, request class, region, and outcome. Avoid high-cardinality labels in the metrics system; join detailed billing records in an analytical store.

Important unit metrics:

- cost per ingested document/page;
- cost per grounded answer;
- cost per completed invoice review;
- cost per reconciled external operation;
- tokens/tool calls per successful workflow;
- retry and failure waste;
- human-review cost and avoided errors;
- gross margin by product tier.

A cheaper model that creates more corrections or escalations can increase total cost.

## Cost controls

Apply controls in order:

1. remove unnecessary work and retries;
2. bound inputs, outputs, retrieval, and workflow steps;
3. cache only version-compatible safe results;
4. batch and schedule delay-tolerant work;
5. route by task difficulty using evaluated policies;
6. right-size compute and databases;
7. use commitments only for predictable baseline;
8. use spot/preemptible capacity only for checkpointable work;
9. archive or delete data under retention policy;
10. set budgets, alerts, and hard stops with safe behavior.

Budgets exist at request, workflow, tenant, and system levels. When a budget is reached, return a truthful partial result, queue for approval, or stop optional work—never silently lower safety controls.

## Multi-tenant fairness

Fairness is a scheduling policy, not merely equal request counts. Tenants vary in plan, urgency, workload size, and purchased guarantees.

Controls include:

- token buckets for rate and burst;
- concurrency limits per tenant and workload;
- weighted fair queuing;
- deficit round robin for variable-size jobs;
- separate priority queues with starvation protection;
- maximum document, context, output, and workflow budgets;
- reserved capacity for critical/reconciliation work;
- global admission control;
- dedicated pools for very large or regulated tenants.

Estimate work before admission: pages, input tokens, requested output tokens, tool count, or historical class. Reconcile estimated versus actual cost afterward.

Avoid FIFO alone. One tenant’s 100,000-page import can block every interactive request. Avoid strict priority without aging: low-priority work may never run.

## Token-bucket example

A tenant receives:

```text
refill rate: 100 work units/minute
burst capacity: 300 units
interactive query: 2–20 units
invoice workflow: 30–100 units
bulk document page: 0.2 units
```

The bucket permits brief bursts while limiting sustained consumption. Global admission still protects the fleet; per-tenant limits alone cannot prevent total overload.

Rate limits should return the limit scope, retry time, and request identifier. Internal retries must consume budget too, or they bypass fairness.

## Noisy-neighbor isolation

A noisy neighbor can exhaust database connections, cache memory, queue workers, provider quotas, or human reviewers.

Use:

- tenant-aware admission and scheduling;
- per-tenant connection/concurrency budgets;
- workload pools for interactive, batch, and reconciliation;
- database query limits and statement timeouts;
- cache namespaces and quotas where needed;
- provider quota partitions;
- tenant-scoped circuit breakers for pathological input;
- shuffle sharding to reduce shared fate.

Dedicated infrastructure improves isolation but raises cost and operational complexity. Use it for contractual, regulatory, or scale reasons—not as the default.

## Construction example

At month-end, several contractors upload invoices simultaneously while a large tenant backfills years of drawings. Interactive questions slow, model costs spike, and ERP submissions approach a vendor quota.

The design:

1. classifies interactive search, invoice review, ERP execution, reconciliation, and bulk ingestion separately;
2. admits each tenant through work-unit token buckets;
3. sends bulk ingestion to a lower-priority pool with aging;
4. reserves connector capacity for approved submissions and reconciliation;
5. scales parsing on weighted page backlog and inference on tokens in flight;
6. caps context/output and uses evaluated model routing;
7. schedules predictable month-end warm capacity;
8. reports queue position and deadlines honestly;
9. attributes token, parsing, storage, connector, and review cost to workflows;
10. sheds optional reindexing before delaying approved financial actions.

If the ERP quota is exhausted, approved actions queue visibly; the system does not spray retries or mark them completed.

## Healthcare variation

Morning rounds create a burst of discharge summaries while a research export starts. Clinical work gets isolated capacity and deadline-aware scheduling; the export is checkpointed and throttled.

Fairness cannot mean equal treatment of an optional export and time-sensitive patient care. Policy defines priority, but authorization and patient isolation remain identical. An expensive patient record is not truncated silently: the system reports missing evidence and requires clinician review.

Cost optimization never routes protected data to an unapproved region/provider or removes required audit, retention, and safety controls.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| High utilization | Better unit economics | Tail latency and weak resilience |
| Large headroom | Handles bursts/failure | Idle cost |
| Reactive autoscaling | Efficient normal load | Cold-start lag |
| Scheduled capacity | Ready for known peaks | Forecast error |
| Shared pool | High utilization | Noisy neighbors |
| Dedicated pool | Isolation | Fragmentation and cost |
| Strict priority | Protects critical work | Starvation |
| Weighted fairness | Controlled sharing | Policy complexity |
| Aggressive batching | Throughput | Interactive latency |
| Longer context | Potential recall | Cost, latency, distraction |
| Cheaper model route | Lower call cost | Corrections/fallback may rise |
| Hard budget | Predictable spend | Partial or rejected work |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Size only by average RPS | Peak collapse | Distributions and scenarios |
| CPU as sole signal | Hidden quota/queue bottleneck | Resource-specific saturation |
| Autoscale after saturation | SLO breach during cold start | Predictive headroom/warm pool |
| Depth without work size | Wrong drain estimate | Weighted backlog and age |
| Unlimited retries | Amplified load/cost | Retry budgets and admission |
| FIFO for all work | Bulk tenant blocks users | Fair workload scheduling |
| Per-tenant limit only | Aggregate overload | Global admission too |
| Strict priority forever | Batch starvation | Aging/minimum shares |
| Cost per model call | Ignores corrections/failures | Cost per successful outcome |
| Fallback without quota | Fails during outage | Reserved tested capacity |
| Cache across versions/tenants | Wrong or leaked result | Versioned tenant-safe keys |
| Cost cut removes safety | Increased harm | Non-negotiable controls |
| Shared metric labels use tenant IDs | Cardinality explosion | Aggregated metrics; billing store |
| Backlog drained at maximum | Downstream re-fails | Controlled drain |

## What to measure

Track arrival rate and work mix; concurrency and service-time distributions; utilization and saturation by bottleneck; queue age, work-weighted backlog, and drain time; autoscaling reaction and cold-start time; rejected/throttled work; retries and wasted work; SLOs by tenant tier and workload; fairness shares and starvation; token/context/output distributions; model/provider quota; cost per successful outcome; idle/headroom and failover readiness; forecast error; and gross margin without exposing individual tenants in operational metrics.

## Interview prompt

> Design capacity, cost, and fairness controls for a multi-tenant AI assistant that ingests construction documents, answers cited questions, reviews invoices, obtains approval, and submits to an ERP. Month-end creates a 5× burst, one tenant starts a 100,000-page backfill, model APIs have token quotas, and ERP writes have a strict rate limit. Adapt the design for healthcare discharge planning.

Cover workload units, estimates, bottlenecks, queueing/headroom, autoscaling, inference capacity, cost allocation, admission, scheduling, tenant isolation, overload behavior, observability, and failure recovery.

## Worked answer

**One-line conclusion:** Size from work-weighted peak demand and failure scenarios, then protect scarce resources with global admission, tenant-aware fair scheduling, isolated priority pools, and budgets measured per successful outcome.

“I would classify interactive search, ingestion, invoice review, ERP execution, and reconciliation because their service demand and deadlines differ. I estimate peak arrival, service-time distributions, retry amplification, and required concurrency, then validate the bottleneck with load, stress, soak, and failure tests. I reserve headroom for burst, autoscaling delay, deployment, and zone/provider failure. APIs scale on concurrency and latency, parsers on weighted page backlog, and inference on active sequences/tokens and memory pressure. Known month-end peaks get scheduled warm capacity. Global admission protects the system; tenant token buckets and weighted fair queues prevent a bulk import from blocking others. Interactive, batch, writes, and reconciliation use bulkheaded pools, with aging to avoid starvation. ERP calls obey vendor quota and approved actions queue visibly. AI requests cap context/output/tool steps and use evaluated routing, batching, and version-safe caching. I track cost per grounded answer or completed workflow, including retries, human corrections, idle resilience, and failures—not just model-call price. In overload I shed experiments and backfills before safety, status, authorization, and reconciliation.”

## Practice lab

Size and schedule this workload:

- normal interactive rate: 25 requests/s; month-end peak: 100/s;
- mean interactive time: 2 seconds, p95: 8 seconds;
- one tenant submits a 100,000-page backfill;
- model quota: 2 million input and 250,000 output tokens/minute;
- ERP limit: 60 writes/minute;
- each invoice averages 12,000 input and 800 output tokens;
- 300 approved invoices arrive within 10 minutes;
- a zone failure removes one-third of inference capacity.

Estimate initial concurrency and headroom, identify the binding limits, define work units and tenant budgets, choose autoscaling signals and pools, calculate whether ERP demand can drain, specify overload behavior, and define five cost/fairness metrics.

## References

- [Google SRE Book — Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Google SRE Workbook — Non-Abstract Large System Design](https://sre.google/workbook/non-abstract-design/)
- [Kubernetes — Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [AWS Builders' Library — Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/)
- [FinOps Foundation — FinOps Framework](https://www.finops.org/framework/)
