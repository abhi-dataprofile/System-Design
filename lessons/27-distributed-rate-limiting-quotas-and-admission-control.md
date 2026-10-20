# 27 — Distributed rate limiting, quotas, and admission control

Previous: [System-design interview drills, scoring rubric, and answer refinement](26-system-design-interview-drills-scoring-and-refinement.md). Study time: 50–60 minutes including the exercise.

## Mental model

Overload control decides **which work may enter**, **how fast it may proceed**, and **who gets scarce capacity**.

```text
demand → identity + cost → policy budgets → admission
                                      ↘ reject / defer / degrade
admitted work → concurrency + queues → dependencies
```

Different controls solve different problems:

- a **rate limit** bounds work per unit time;
- a **quota** bounds work or spend over a longer period;
- a **concurrency limit** bounds simultaneous in-flight work;
- a **queue bound** limits deferred work and maximum waiting;
- **admission control** decides whether a request should enter at all;
- **fair scheduling** allocates capacity among tenants or priorities;
- **load shedding** removes lower-value work when capacity is exhausted.

> Measure requests in the resource they actually consume, reserve capacity for safety-critical work, and reject early enough that overload does not become a cascading failure.

A request-per-second limit is insufficient when one request answers from cache and another parses 500 pages or consumes 100,000 model tokens.

## When to use these controls

Use overload controls at:

- public and internal APIs;
- authentication and expensive query endpoints;
- uploads, OCR, parsing, embedding, and reindexing;
- model gateways with token and provider quotas;
- queues and background workers;
- ERP, email, payment, project-management, and clinical connectors;
- review queues with limited human capacity;
- databases, search clusters, and caches;
- tenant onboarding, exports, and backfills.

Apply protection before the constrained resource. A limit after a model call cannot protect the model budget.

Use multiple layers because each layer knows different facts:

- edge knows IP, account, and broad abuse signals;
- application knows tenant, endpoint, operation class, and business value;
- model gateway knows tokens and model capacity;
- worker knows queue age and downstream quota;
- connector knows vendor-specific limits and reconciliation priority.

Do not rely on client-side throttling for protection.

## Define the constrained resource

Start with a capacity inventory:

| Resource | Better work unit |
| --- | --- |
| API CPU | weighted request cost |
| Database | queries, rows scanned, connections |
| File ingestion | pages, bytes, parser-seconds |
| Embeddings | input tokens or chunks |
| Generation | input/output tokens, GPU-seconds |
| Search | candidate count and query complexity |
| ERP API | vendor calls and concurrent requests |
| Human review | reviewer-minutes by skill |
| Email/export | recipients, bytes, jobs |
| Backfill | source rows plus downstream writes |

For each resource define:

- sustainable and burst capacity;
- hard external quota;
- per-tenant entitlement;
- priority classes;
- deadline and maximum queue age;
- fallback or degraded mode;
- rejection semantics;
- owner and alert.

Capacity changes over time. Provider quotas, reviewer shifts, database health, and regional loss can all reduce the current budget.

## Rate, quota, concurrency, and queueing

These controls are complementary.

### Rate

Protects throughput over time. Example: 100 ERP calls/minute.

### Quota

Protects finite allocation or cost. Example: 20 million model tokens/tenant/month.

### Concurrency

Protects resources whose cost depends on work already in flight. Example: only eight simultaneous OCR jobs for large drawings.

Little’s Law connects concurrency, rate, and latency:

[
L = lambda W
]

If admitted rate is 40 requests/s and average dependency time is 0.5 s, expected in-flight work is about 20. Tail latency and bursts require headroom.

### Queue bound

Absorbs a temporary burst, not permanent overload. If arrival rate remains greater than service rate, the backlog grows without bound.

Estimate drain time:

[
	ext{drain time} =
rac{	ext{queued work}}{	ext{service capacity} - 	ext{new admitted load}}
]

when service capacity exceeds continuing arrivals.

A bounded queue converts invisible latency into an explicit reject/defer decision.

## Algorithms

## Fixed window counter

Counts requests in a wall-clock interval.

Advantages:

- simple;
- cheap;
- easy to explain.

Problem: a client can use the full limit at the end of one window and again at the start of the next, creating a boundary burst.

Use for coarse daily/monthly quotas or low-risk controls, not precise burst protection.

## Sliding log

Stores each request timestamp and counts those in the active window.

Advantages:

- accurate.

Costs:

- memory and update overhead;
- expensive at high cardinality.

Useful for small, high-value identity sets.

## Sliding-window counter

Approximates a rolling window by combining current and previous buckets.

Advantages:

- smoother than fixed windows;
- less state than a full log.

Cost: approximate and clock-sensitive.

## Token bucket

Tokens refill at rate (r) up to burst capacity (b). Each request consumes tokens based on cost.

[
	ext{tokens}(t) =
min(b, 	ext{tokens}(t_0) + r(t-t_0)) - 	ext{cost}
]

Allows bounded bursts while enforcing long-term rate. It is a strong default for APIs and weighted work.

## Leaky bucket

Requests enter a bounded bucket and leave at a steady rate.

It smooths downstream traffic but introduces queueing. When full, new work is rejected or deferred elsewhere.

Use when a fragile vendor requires an even call rate.

## Concurrency limiter

Acquires a permit before work and releases it on completion, timeout, or cancellation. Adaptive variants adjust the limit from latency or error signals.

Use a lease or process-safe cleanup so crashed workers do not permanently consume permits.

## Hierarchical budgets

Real systems need nested constraints:

```text
global
  → region
    → dependency/model
      → tenant
        → user/workflow
          → operation class
```

A request is admitted only if all required budgets allow it.

Example:

- global model tokens/minute;
- model-provider deployment quota;
- tenant contracted tokens/minute;
- user interactive burst;
- workflow remaining budget.

Hierarchies prevent one tenant from consuming the entire global pool while allowing controlled borrowing of unused capacity.

Avoid multiplying strict limits until normal traffic is accidentally rejected. Test effective capacity at every path.

## Weighted costs

Assign cost based on expected resource consumption:

```text
cost = base
     + page_weight × pages
     + input_token_weight × tokens
     + output_token_reservation
     + tool_weight × external calls
```

Reserve expected maximum cost before admission, then refund unused capacity where accounting supports it.

For model generation, input tokens are known but output tokens are bounded rather than known. Reserve an output allowance. Do not allow unlimited generation after passing a one-request limit.

Prevent clients from understating cost: the server computes pages, tokens, query complexity, and tool class.

Weights need calibration from observed CPU, latency, GPU time, and downstream usage. They are policy approximations, not physical truth.

## Distributed enforcement choices

## Central authoritative limiter

Every request consults a strongly consistent or atomic counter service.

Benefits:

- accurate global view;
- simple policy semantics.

Costs:

- added latency;
- availability dependency;
- hot keys;
- cross-region coordination.

Use for scarce, expensive global capacity or billing-sensitive quotas.

## Local limiters

Each instance receives a share of the budget.

Benefits:

- fast;
- survives central-service latency;
- no request-time network call.

Costs:

- aggregate overshoot;
- uneven utilization;
- rebalancing when instances change.

If (N) instances each allow the full tenant limit, the effective limit becomes (N) times too large. Allocate per-instance shares or use leased token batches.

## Leased token batches

A central service grants a limited batch of tokens to edge or regional instances. Instances admit locally until the lease/batch expires.

Benefits:

- low request latency;
- bounded overshoot;
- regional resilience.

Costs:

- stranded tokens;
- lease complexity;
- temporary unfairness.

Choose lease size from acceptable overshoot and refill latency.

## Partitioned ownership

A tenant or key maps to one rate-limit shard.

Benefits:

- atomic per-key updates;
- good horizontal scale.

Costs:

- hot tenants;
- shard movement;
- regional failover semantics.

Use consistent hashing plus replication, but define which replica can spend the budget.

## Approximate/global reconciliation

Regions enforce local limits and periodically reconcile usage.

Use for soft analytics or cost budgets where bounded overshoot is acceptable. Do not use when a vendor hard-rejects above a shared quota or safety requires a strict cap.

## Consistency and failure policy

Decide per limiter whether to fail open, fail closed, or use a local emergency budget.

- **Fail open:** preserves availability but risks overload or cost.
- **Fail closed:** protects scarce capacity but may cause outage.
- **Emergency lease:** allows bounded critical work using last-known allocation.

Examples:

- static page: fail open;
- login abuse limit: often local conservative fallback;
- expensive export: fail closed or defer;
- clinical operational read: reserved emergency capacity;
- autonomous external write: hold if policy/admission state is unavailable;
- reconciliation of outcome-unknown payment: use protected capacity.

Do not silently convert a failed limiter into unlimited access.

Clock skew matters for time windows and leases. Prefer monotonic elapsed time locally and server-assigned lease expiration where possible.

## Admission control before queues

A queue is not capacity. It is delayed demand.

Admission should consider:

- estimated work cost;
- current concurrency;
- queue age and drain time;
- request deadline;
- tenant entitlement;
- dependency health and quota;
- priority;
- whether a degraded alternative exists.

Reject work that cannot finish before its deadline. Accepting it only to time out later wastes resources and increases tail latency.

For asynchronous work, return a durable operation ID and an estimated state such as queued, scheduled, deferred, or rejected. Never promise a completion time without evidence.

## Priority and fairness

Common priority classes:

1. safety/security controls;
2. reconciliation of uncertain external outcomes;
3. interactive authorized user requests;
4. time-sensitive workflow actions;
5. ordinary background ingestion;
6. reindexing, analytics, and backfills.

Use weighted fair queuing, deficit round robin, or per-tenant queues with a fair scheduler. Aging prevents starvation.

Reserve capacity rather than relying only on priority. Under saturation, low-priority work may already occupy every worker or connection.

Per-tenant minimums protect small tenants. Maximums constrain noisy neighbors. Controlled borrowing improves utilization when other tenants are idle.

Fairness should use cost units, not message counts. One 500-page drawing should not equal one invoice.

## Retries and amplification

If a dependency slows, uncoordinated retries multiply demand.

Use:

- deadline propagation;
- bounded attempts;
- exponential backoff with jitter;
- retry budget as a fraction of normal traffic;
- idempotency;
- circuit breakers;
- hedging only for safe reads and within a budget;
- `Retry-After` or equivalent hints;
- client-side token buckets in addition to server protection.

A server returning 429 while every client retries simultaneously after one second creates a synchronized storm. Add jitter and use absolute deadlines.

Separate rate-limit rejection from transient dependency failure and permanent validation failure.

## HTTP and asynchronous responses

For HTTP overload:

- use `429 Too Many Requests` for client/identity quota;
- use `503 Service Unavailable` when service capacity is temporarily unavailable;
- include `Retry-After` when a useful retry time is known;
- expose stable machine-readable reason codes;
- return limit, remaining budget, and reset metadata only when safe;
- avoid leaking other tenants’ usage or internal capacity.

For queued work:

- `202 Accepted` means durably accepted, not completed;
- provide operation status;
- define cancellation;
- expire work that can no longer meet its purpose;
- make submission idempotent.

A rejected request should not consume the full operation quota, but abuse of rejected requests still needs edge protection.

## AI inference admission

AI work has several bottlenecks:

- provider requests/minute;
- tokens/minute;
- context length;
- GPU memory;
- concurrent sequences;
- batch formation;
- tool-call quotas;
- cost budget.

Admission can route:

- high-risk complex request → capable model;
- simple extraction → smaller model;
- repeated safe query → scoped cache;
- low-priority summarization → asynchronous batch;
- overload → evidence-only or deterministic response;
- budget exhausted → defer, ask for narrower scope, or require explicit override.

Do not degrade to an unapproved model for sensitive or high-risk data. Fallback must preserve region, privacy, capability, and evaluation requirements.

Preemption may waste generated tokens. Prefer separate pools or queues for interactive and batch work.

## Vendor connector quotas

Treat vendor limits as shared capacity.

Maintain:

- per-credential and per-tenant quota;
- global vendor quota;
- endpoint-specific weights;
- bounded concurrency;
- quota reset and `Retry-After`;
- priority for reconciliation;
- request deadline;
- stable operation ID.

Never let a historical backfill consume the calls needed to establish whether a timed-out ERP write succeeded.

When multiple workers share one credential, enforce the quota collectively or distribute leased capacity. Log normalized quota decisions without credentials or sensitive payloads.

## Abuse prevention versus fairness

Rate limiting is one layer of abuse prevention, not a complete defense.

Signals may include:

- identity and credential reputation;
- IP/network;
- device/session;
- endpoint and action;
- payload size and complexity;
- repeated authorization failures;
- impossible navigation or automation patterns.

Avoid making IP address the only identity; offices, hospitals, and job sites share networks. Apply account/tenant/action controls and investigate false positives.

Security limits should not reveal whether a resource or account exists. Use generic responses where enumeration is a risk.

## Construction example

Month-end creates:

- 5× invoice uploads;
- a 100,000-page document backfill;
- limited OCR and model tokens;
- an ERP quota of 120 calls/minute;
- interactive questions and approval writes;
- several outcome-unknown writes awaiting reconciliation.

Design:

1. edge limits abusive requests by identity and tenant;
2. upload admission reserves page-weighted ingestion units;
3. tenant queues and weighted scheduling isolate the backfill;
4. interactive invoice evidence has a reserved worker pool;
5. model gateway charges estimated input/output tokens;
6. ERP token bucket smooths to the vendor rate;
7. reconciliation owns a protected ERP-call reserve;
8. approval writes outrank backfill reads but retain idempotency;
9. queue-age admission rejects/defer work that cannot meet deadline;
10. degraded mode returns structured/cached evidence without model synthesis;
11. dashboards show tenant work units, queue drain time, throttling, and protected-capacity use.

## Healthcare operations variation

A hospital operational platform receives a morning surge of discharge and scheduling work while a regional model provider is constrained.

Interactive patient-scoped operational reads and time-sensitive reconciliation receive reserved capacity. Analytics, historical reprocessing, and low-urgency summarization defer. Clinical-role authorization is never bypassed to improve throughput.

Limits use facility, authenticated workforce identity, workflow class, and cost—not IP alone. PHI is not copied into limiter keys or error messages. If admission or policy authority is unavailable, consequential clinical operations follow an approved manual path; the system does not route sensitive work to an unapproved fallback model.

Emergency access is a governed priority class with reason capture, narrow duration, monitoring, and retrospective review—not a universal bypass token.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Strict global limiter | Accurate budget | Latency and availability coupling |
| Local limiter | Fast and resilient | Bounded overshoot |
| Token bucket | Controlled bursts | State/refill tuning |
| Leaky bucket | Smooth dependency traffic | Added waiting |
| Request-count units | Simple | Unfair for variable work |
| Weighted units | Better resource match | Calibration complexity |
| Queue burst | Absorbs short spike | Latency and memory |
| Early rejection | Protects system | Lost/deferred user work |
| Fail open | Availability | Overload/cost/security risk |
| Fail closed | Protects capacity | User-visible outage |
| Reserved capacity | Critical work survives | Lower average utilization |
| Shared tenant pool | Efficient | Noisy-neighbor risk |
| Per-tenant hard quota | Isolation | Stranded capacity |
| Borrowing | Better utilization | More policy complexity |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Limit only by request count | Expensive work dominates | Weighted server-computed cost |
| Limit after expensive work | Resource already consumed | Admit before dependency |
| One local full limit per instance | (N)-fold overshoot | Shares, sharding, or token leases |
| Central limiter is hot/unavailable | Added outage | Shard, cache leases, explicit fallback |
| Queue is unbounded | Memory/latency collapse | Bound by work and deadline |
| Retry ignores 429 | Amplification storm | Backoff, jitter, retry budget |
| All capacity goes to backfill | Interactive/reconciliation starvation | Separate pools and reserves |
| Priority without preemption/reserve | Critical work still waits | Reserved capacity |
| Fixed window boundary burst | Dependency spike | Token/sliding bucket |
| Permit leaks after crash | Artificial capacity loss | Lease/timeout cleanup |
| Tenant missing from limiter key | Cross-tenant interference | Trusted tenant-scoped hierarchy |
| Client declares its cost | Quota evasion | Server measurement |
| Fail open silently | Unlimited spend/load | Audited bounded emergency policy |
| Limit metadata leaks capacity | Security/tenant information leak | Minimal scoped headers |
| Clock skew breaks reset | Early/late admission | Monotonic time and leases |
| Fallback model violates policy | Privacy/safety incident | Approved capability-aware routing |
| Reconciliation is throttled | Unknown external outcomes persist | Protected quota |
| Emergency priority is permanent | Fairness and governance failure | Expiring audited override |

## What to measure

Track admitted, rejected, deferred, and degraded work by reason; work units per tenant and operation; token/permit utilization; concurrency; queue age and drain time; fairness and starvation; provider/vender quota headroom; 429/503 rate; retry amplification; limiter latency and availability; overshoot; stranded leases; emergency-budget use; cost per successful outcome; outcome-unknown age; and false-positive support cases.

Alert before a hard vendor quota is exhausted and when backlog drain time exceeds business deadlines.

## Interview prompt

> Design distributed rate limiting, quotas, and admission control for a multi-tenant AI operations platform. Work includes interactive queries, document ingestion, model inference, background reindexing, human-review tasks, and ERP writes. One tenant may generate 40% of traffic; model and ERP providers enforce shared quotas; month-end load is 5×; and uncertain ERP outcomes require reconciliation.

Cover algorithms, weighted units, hierarchical budgets, distributed consistency, queues, fairness, retries, HTTP semantics, AI routing, vendor quotas, failure policy, observability, and security.

## Worked answer

**One-line conclusion:** Admit work using hierarchical, cost-weighted budgets; reserve capacity for interactive and reconciliation paths; and reject or degrade early enough to prevent queues and retries from turning scarcity into a cascading failure.

“I would inventory each scarce resource and choose a work unit: pages for ingestion, tokens for models, calls/concurrency for ERP, and reviewer-minutes for human queues. Edge controls handle broad abuse, while application admission uses trusted tenant, operation class, estimated cost, deadline, and dependency health.

Token buckets allow bounded interactive bursts; concurrency leases protect parsers, databases, and model deployments; bounded queues absorb only temporary spikes. Budgets are hierarchical across global, region, provider, tenant, user, and workflow. Small tenants receive minimum shares, noisy tenants have maxima, and unused capacity can be borrowed. Interactive evidence, safety controls, and reconciliation use reserved pools; backfills and reindexing run in lower-priority lanes with aging.

For strict shared ERP quotas, workers use an authoritative or leased-token limiter with endpoint weights and stable operation IDs. A 429 or `Retry-After` drives jittered backoff within a retry budget. Admission rejects work that cannot finish before deadline. If the limiter fails, safe reads may use a bounded emergency lease; expensive jobs defer; consequential writes hold. Model routing may choose an approved smaller model or evidence-only response, never an unapproved privacy fallback.

I would monitor work-unit demand, admitted/rejected/degraded outcomes, concurrency, queue drain time, tenant fairness, provider headroom, retry amplification, limiter latency, overshoot, and reconciliation age.”

## Practice lab

Design overload control for:

- 80 tenants;
- one tenant produces 40% of pages;
- month-end traffic is 5×;
- OCR supports 200 page-equivalents/minute;
- the model provider allows both requests/minute and tokens/minute;
- ERP allows 120 calls/minute across shared credentials;
- 12 writes have unknown outcomes;
- a 100,000-page backfill is running;
- the central limiter becomes unavailable;
- interactive users need a bounded response deadline.

Specify work units, token/concurrency/queue controls, hierarchical keys, distributed enforcement, overshoot tolerance, priority/reserved pools, admission decisions, retry semantics, degraded modes, HTTP/async responses, and metrics.

## References

- [RFC 6585 — Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9331 — RateLimit Fields for HTTP](https://www.rfc-editor.org/rfc/rfc9331)
- [Google SRE Book — Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Envoy — Global rate limiting](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/global_rate_limiting)
