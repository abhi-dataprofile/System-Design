# 30 — Service discovery, health checking, and resilient routing

Previous: [Sharding, partitioning, rebalancing, and hot-key mitigation](29-sharding-partitioning-rebalancing-and-hot-key-mitigation.md). Study time: 50–60 minutes including the exercise.

## Mental model

Service discovery answers **where a dependency can be reached**. Health checking estimates **whether an endpoint should receive a class of traffic**. Resilient routing decides **which eligible endpoint to choose, for how long, and what to do when reality changes**.

```text
logical service
  → discovered endpoints + metadata
    → eligible subset
      → routing policy
        → bounded attempt with deadline
          → observed outcome feeds health
```

> Discovery data is a hint about possible destinations; request outcomes are evidence about whether a destination can serve this request now.

A registered endpoint may be alive but not ready, ready for reads but not writes, healthy for one tenant yet unable to reach that tenant's shard, or operational but incompatible with a request's protocol version. “Healthy” is not one universal Boolean.

Keep these concerns separate:

| Concern | Question |
| --- | --- |
| Naming | What logical dependency does the caller need? |
| Discovery | Which endpoints advertise that dependency? |
| Eligibility | Which endpoints meet region, version, identity, and capability constraints? |
| Health | Which eligible endpoints appear able to serve? |
| Balancing | How is traffic distributed among them? |
| Resilience | What happens on slowness, failure, or stale information? |
| Authorization | Is this caller allowed to invoke this operation? |

Discovery never replaces authorization. Knowing an endpoint address is not permission to call it.

## When to use discovery

Use dynamic discovery when endpoints change through:

- autoscaling;
- container or virtual-machine replacement;
- rolling deployments;
- zone or region failover;
- shard movement;
- model deployment changes;
- tenant-dedicated services;
- blue/green or canary releases;
- external provider failover.

Static configuration is reasonable for a small, stable dependency set when operational updates are controlled. Dynamic discovery adds a control plane, caching, convergence, and failure modes. Do not introduce it only to imitate a microservice architecture.

Before choosing a mechanism, define:

- endpoint churn rate;
- acceptable discovery staleness;
- region and data-residency constraints;
- protocol and API-version compatibility;
- request classes and health semantics;
- expected behavior during control-plane loss;
- connection lifetime and drain requirements;
- security identity and certificate rotation;
- whether routing occurs at the client, proxy, gateway, or platform.

## Discovery mechanisms

### DNS-based discovery

A logical name resolves to one or more addresses, sometimes through SRV records.

Benefits:

- ubiquitous client support;
- simple operational model;
- independent of application language.

Costs:

- resolver and application caches may ignore intended TTL behavior;
- limited endpoint metadata;
- long-lived connections outlive DNS answers;
- negative caching delays recovery;
- DNS success does not prove application readiness.

DNS works well for stable service front doors. It needs connection recycling, health-aware load balancing, and careful TTL testing when endpoints change frequently.

### Registry-based discovery

Instances register with a service registry or are derived from an orchestrator's authoritative state. Callers or proxies watch/query endpoints with metadata.

Benefits:

- richer capability, zone, version, and tenant metadata;
- faster endpoint updates;
- explicit leases/session semantics.

Costs:

- registry becomes a control-plane dependency;
- watches can disconnect or miss compaction windows;
- stale local caches remain possible;
- registration does not equal readiness.

After a watch reconnects, obtain a fresh snapshot before applying later changes. Treat watch events as notifications, not an infallible event log.

### Platform-native discovery

Orchestrators such as Kubernetes maintain service endpoints from workload and readiness state. A stable virtual service or headless service exposes eligible pods.

Benefits:

- integrated lifecycle and rollout behavior;
- fewer custom components.

Costs:

- probes and endpoint propagation still need correct design;
- kube-proxy, DNS, service mesh, and application caches can converge at different speeds;
- external and cross-cluster routing require additional policy.

### Configuration or directory routing

A versioned directory maps tenants, shards, models, or providers to endpoints.

Use when placement has business constraints that generic discovery cannot infer: data residency, dedicated tenants, model approval, or shard ownership. The directory should publish immutable versions/epochs, and destinations should reject stale ownership when correctness depends on it.

## Registration lifecycle

An endpoint should not receive traffic merely because its process started.

A safe lifecycle:

1. process starts unregistered or not-ready;
2. dependencies and configuration initialize;
3. startup checks confirm the instance can become useful;
4. readiness becomes true for supported traffic classes;
5. endpoint enters discovery;
6. during shutdown, readiness becomes false first;
7. routers stop assigning new requests;
8. existing requests and streams drain within a bound;
9. the endpoint deregisters and exits;
10. forced termination occurs only after the grace period.

Registration should carry instance identity, service version, protocol/capabilities, region, zone, deployment, and possibly supported request classes. Do not store secrets or tenant-sensitive details in broadly readable discovery metadata.

Use leases or orchestrator ownership so crashed endpoints eventually disappear. Explicit deregistration accelerates planned shutdown but cannot be the only cleanup mechanism.

## Startup, liveness, and readiness

### Startup check

Answers: “Has initialization completed enough for normal probes to be meaningful?”

Use it for slow model loading, migrations, cache warming, or large configuration fetches. Without a startup allowance, liveness may repeatedly kill a process that would eventually become healthy.

### Liveness check

Answers: “Is this process irrecoverably stuck such that restarting may help?”

Keep it narrow: event-loop progress, fatal deadlock, or internal watchdog failure. Do not fail liveness whenever a remote dependency is down. Restarting every caller during a database outage creates a restart storm and destroys useful diagnostics.

### Readiness check

Answers: “Should this endpoint receive new traffic of this kind now?”

Readiness may depend on configuration loaded, server accepting connections, required local state, and critical dependency reachability. Avoid one overly strict probe that removes every endpoint because a noncritical dependency is degraded.

Where request classes differ, expose capabilities or separate pools:

- read-ready;
- write-ready;
- streaming-ready;
- tool-execution-ready;
- model-version-ready.

A service may remain ready for cached evidence reads while refusing ERP writes.

## Active and passive health

**Active checks** send synthetic probes at an interval. They detect silent failures before user traffic but add load and can give false confidence if the probe path is shallow.

**Passive checks** learn from real requests: connection errors, timeouts, response codes, latency, and protocol failures. They reflect real behavior but need enough traffic and careful classification.

Combine them:

- active readiness for baseline eligibility;
- passive outlier detection for local, recent evidence;
- dependency-wide circuit state for correlated failures;
- synthetic end-to-end checks for critical workflows.

Classify failures. A caller authorization error, invalid invoice, or model safety refusal is not endpoint unhealthiness. Connection refusal, repeated deadline exhaustion, and protocol corruption may be.

Use rolling windows, minimum sample sizes, and ejection bounds. One slow request should not permanently exile an instance.

## Client-side versus server-side routing

### Client-side

The caller receives endpoints and selects one.

Benefits:

- no extra network hop;
- caller sees request-specific constraints;
- fast local reaction to failures.

Costs:

- routing logic duplicated across languages;
- uneven policy rollout;
- every client handles discovery, pooling, retry, and security correctly.

### Proxy or gateway

A local sidecar, node proxy, service mesh, or centralized gateway selects endpoints.

Benefits:

- consistent transport policy;
- centralized observability and mTLS;
- easier outlier detection and gradual rollout.

Costs:

- another component and failure mode;
- extra hop or resource consumption;
- application semantics may be invisible to the proxy.

Use application logic for business-aware routing—tenant residency, model approval, idempotency—and transport infrastructure for connection balancing, mTLS, and bounded outlier ejection. Do not hide unsafe retries behind a generic proxy.

## Load-balancing policies

| Policy | Good fit | Limitation |
| --- | --- | --- |
| Round robin | Similar instances and request cost | Ignores in-flight work |
| Least requests | Variable request duration | Needs current load signal |
| Weighted round robin | Unequal capacity or rollout weight | Static weights can become stale |
| Random/two choices | Large fleets, low coordination | Approximate |
| Consistent hash | Session/cache affinity | Hot keys and remapping |
| Locality-aware | Multi-zone/region traffic | Can overload surviving locality |
| Priority failover | Explicit primary/secondary | Failback and cold capacity |
| EWMA latency | Avoid recently slow endpoints | Feedback can chase noise |

Balance on work, not only request count. One OCR job or long generation may cost far more than a metadata read. Concurrency-aware routing often beats simple round robin for heterogeneous AI work.

Session affinity should be a performance optimization, not a correctness requirement. Durable workflow state belongs in an authoritative store so another instance can resume.

## Locality and failure domains

Prefer same-zone traffic when it lowers latency and cross-zone cost, but preserve zone independence.

A routing hierarchy might be:

1. eligible endpoints in the same zone;
2. other zones in the approved region;
3. another approved region only if policy, consistency, and failover state permit;
4. safe degraded behavior.

Reserve cross-zone/region capacity. If one zone fails and all traffic shifts, remaining zones need headroom or admission control.

Do not retry blindly across regions after an uncertain write. Region failover for reads differs from failover for approval, payment, or ERP actions. Consequential writes need authoritative ownership, fencing, idempotency, and reconciliation.

## Deadlines, retries, and routing

A retry is a new load-generating routing decision.

Propagate an end-to-end deadline. Each attempt receives only the remaining budget:

```text
request deadline
  - queue time
  - completed attempts
  - response/serialization margin
  = budget for next attempt
```

Use:

- bounded attempts;
- exponential backoff with jitter;
- per-call and end-to-end timeouts;
- retry budgets;
- idempotency keys for safe repeated operations;
- outlier ejection;
- circuit breakers;
- concurrency/admission limits.

Retry connection failures differently from application failures. Avoid retrying validation, authorization, conflict, or safety refusals.

Hedged requests can reduce read tail latency by launching a second attempt after a percentile delay, but they increase load. Use them only for idempotent reads, within a hedge budget, and cancel losers. Never hedge an ERP write.

Retries across layers multiply. If SDK, sidecar, gateway, and service each retry three times, one request can create dozens of attempts. Assign retry ownership to one layer for each boundary.

## Circuit breakers and outlier ejection

An **outlier detector** removes one endpoint that behaves worse than peers. A **circuit breaker** stops calls to a dependency or operation class when failure is systemic.

Circuit states often include closed, open, and half-open. Half-open sends a small number of probes before restoring traffic.

Avoid ejecting every endpoint at once. Cap the ejected fraction and combine ejection with overload control. If failures are correlated because the database is down, cycling through every service instance only amplifies load.

Circuit state should be scoped appropriately:

- endpoint;
- zone;
- service;
- tenant/credential;
- operation;
- external provider.

An ERP credential can be rate-limited or invalid for one tenant while the provider remains healthy for others.

## Slow failure and gray failure

Many incidents are not clean crashes:

- endpoint accepts TCP but its worker pool is exhausted;
- one availability zone cannot reach a database;
- only large payloads time out;
- one model version returns malformed tool arguments;
- one tenant's credential is invalid;
- a service is healthy internally but DNS routes some callers to stale addresses.

Use request-class metrics, dependency reachability, saturation, and synthetic workflows. A shallow `200 OK` probe cannot detect these.

Health must not create a feedback loop: high latency causes ejection, traffic concentrates on fewer endpoints, they slow down, and the fleet collapses. Pair ejection with concurrency limits, load shedding, gradual recovery, and maximum ejection.

## Graceful draining and deployment

During a rollout:

1. mark the endpoint not-ready;
2. publish removal from discovery;
3. wait for propagation and load-balancer convergence;
4. stop accepting new work;
5. finish or checkpoint bounded requests;
6. close idle keep-alive connections;
7. notify or resume long streams/workflows safely;
8. terminate after the grace period.

Long-lived HTTP/2, WebSocket, or streaming model connections can keep sending work after endpoint removal. Send graceful shutdown signals, enforce maximum connection age, and make reconnection resumable.

For queued jobs, discovery should resolve at execution time. Do not serialize a transient pod address into durable work.

Canary endpoints need explicit weights and compatible request selection. Keep sticky assignment stable enough to measure the canary, but provide rapid disable and ensure high-risk operations remain outside the canary until evidence permits them.

## Discovery control-plane failure

Define behavior before the registry or DNS control plane fails.

A reasonable policy:

- continue with the last known valid endpoint set for a bounded stale period;
- keep passive health and endpoint ejection working locally;
- refuse newly discovered capabilities or regions without authoritative metadata;
- preserve residency and identity constraints;
- avoid infinite stale operation;
- alert on cache age and refresh failure;
- fail closed for consequential routing when ownership or policy is unknown.

Do not interpret an empty discovery response as “delete every endpoint” without version and consistency checks. Preserve the last complete snapshot during an interrupted update, while honoring known endpoint failures.

Bootstrap dependencies carefully. The discovery system itself may need DNS, certificates, storage, and monitoring. Document how it starts and recovers.

## Security and tenant isolation

Secure discovery and routing with:

- authenticated workload identity;
- mTLS between services where appropriate;
- authorization based on service identity and operation;
- signed or authenticated metadata;
- least-privilege registry reads/writes;
- certificate rotation without traffic loss;
- tenant/region constraints from trusted context;
- sanitized endpoint and error metadata;
- audit of routing-policy and failover changes.

Never trust a caller-supplied tenant ID to select a privileged or regional route. Derive tenant context from authenticated identity and verify it at the destination too.

Service-mesh mTLS protects transport but does not implement business authorization. A valid service identity may still lack permission to approve an invoice or read a patient record.

## External and AI-provider routing

External APIs and model providers need a capability-aware gateway.

Route using:

- approved region and data-handling policy;
- model/provider capability;
- tenant contract;
- health, latency, quota, and cost;
- input sensitivity and risk;
- context length and modality;
- evaluation-approved model version;
- remaining deadline.

A fallback is not safe merely because it responds. It must satisfy privacy, residency, capability, and evaluation requirements. Record the chosen provider/model/configuration for lineage.

Provider health should distinguish authentication, quota, rate limit, model overload, safety refusal, malformed response, and network failure. A 429 should influence admission and retry timing, not mark every provider endpoint dead.

For streaming generation, handle mid-stream failure explicitly. Automatic replay may duplicate tokens or tool calls. UI can mark an incomplete response and offer a safe retry; workflows should persist state and never execute a tool merely because partial model output suggested it.

## Construction operations example

A multi-tenant construction assistant has API, document, retrieval, model-gateway, workflow, and ERP-connector services across three zones. Month-end creates 5× traffic, one zone loses database connectivity, and a new connector version is rolling out.

Design:

1. workloads register through the platform only after startup and readiness checks;
2. metadata includes zone, version, protocol, and supported request classes;
3. routers prefer same-zone endpoints but reserve regional headroom;
4. passive health ejects connector instances that cannot reach the ERP, with a bounded ejection fraction;
5. evidence reads can route cross-zone inside the approved region;
6. invoice execution stays with the authoritative workflow/operation record;
7. retries use one owner, stable operation IDs, deadlines, jitter, and budgets;
8. a timed-out ERP write becomes `outcome_unknown`, never a blind route to another connector;
9. the canary connector receives low-risk read/reconciliation traffic before approval writes;
10. draining removes readiness, waits for propagation, checkpoints work, and closes long-lived connections;
11. during discovery outage, routers use a bounded last-known set but do not invent a new region or shard owner;
12. the model gateway falls back only to evaluated, contract- and residency-compatible models;
13. dashboards separate endpoint failure, zone dependency failure, provider quota, tenant credential failure, and application rejection.

The important distinction is that routing can find another connector, but only the durable workflow can decide whether another ERP attempt is safe.

## Healthcare operations variation

A healthcare operations platform routes FHIR reads, document retrieval, scheduling workflows, and approved model calls across zones. Endpoint eligibility includes organization, region, capability, clinical integration version, and workload identity.

A service that cannot reach one hospital's FHIR endpoint is not globally unhealthy. Health and circuit state are scoped to organization, credential, and operation. PHI stays within the approved region; a lower-latency endpoint elsewhere is ineligible.

During partial failure, the platform may serve policy-approved cached operational data with visible freshness, while consequential actions use current authorization, authoritative record versions, and human workflows. A model fallback must be approved for PHI and the specific task. Discovery or mesh access never grants clinical authorization.

Probe payloads, logs, and health labels contain no PHI. Synthetic checks use test tenants and records. During control-plane loss, the system preserves last-known safe routes only for a bounded interval and follows approved manual procedures when ownership or policy becomes uncertain.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| DNS discovery | Simple and universal | Cache and metadata limitations |
| Registry/watch | Rich, timely endpoints | Control-plane dependency |
| Client-side routing | Request-aware and no proxy hop | Duplicated complexity |
| Proxy/mesh routing | Consistent transport policy | Extra component and semantic blindness |
| Active probes | Detect idle endpoint failure | Synthetic path may mislead |
| Passive health | Reflects real traffic | Needs samples; can punish rare errors |
| Strict readiness | Protects callers | Cascading removal during dependency outage |
| Narrow readiness | Preserves availability | Some requests still fail |
| Same-zone preference | Low latency/cost | Surviving-zone overload |
| Cross-region fallback | Availability | Consistency, cost, residency risk |
| Least requests | Handles variable durations | Requires accurate in-flight state |
| Consistent hashing | Cache/session locality | Hot keys and remapping |
| Outlier ejection | Avoids bad instances | Feedback-loop collapse |
| Long discovery cache | Survives control-plane outage | Stale endpoints |
| Short cache | Fast convergence | Control-plane load/coupling |
| Generic proxy retry | Easy transport recovery | Duplicate unsafe operations |
| Capability-aware gateway | Safe model/provider selection | Policy and evaluation complexity |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Registered means healthy | Traffic reaches unready process | Startup and readiness gates |
| Remote dependency in liveness | Restart storm | Keep liveness local and narrow |
| One Boolean health for all work | Useful capacity removed or unsafe work sent | Class-specific readiness/capability |
| DNS changes but pools persist | Traffic hits retired endpoint | Max connection age and drain |
| Watch reconnect misses state | Stale endpoint set | Fetch fresh versioned snapshot |
| Empty partial update replaces full set | Total outage | Atomic snapshots and version checks |
| Every layer retries | Attempt explosion | One retry owner and budget |
| Retry ignores deadline | Work finishes after usefulness | Propagate remaining deadline |
| Write retried to another endpoint | Duplicate external effect | Idempotency and reconciliation |
| All slow endpoints ejected | No capacity, cascading failure | Ejection cap and admission control |
| Same-zone routing after zone degradation | Concentrated failure | Dependency-aware locality failover |
| Cross-region route ignores policy | Residency breach | Hard eligibility constraints |
| Health treats 4xx as endpoint failure | Healthy fleet ejected | Classify outcomes |
| Probe is too shallow | Gray failure missed | Passive metrics and synthetic workflow |
| Probe is too deep | Shared outage removes all endpoints | Separate dependency/capability health |
| Shutdown before convergence | Requests reset | Drain period and propagation budget |
| Durable job stores pod IP | Job targets dead instance | Resolve logical service at execution |
| Fallback model is unevaluated | Privacy or quality failure | Approved capability matrix |
| Discovery outage uses stale data forever | Calls retired/compromised endpoints | Bounded stale TTL and fail-safe mode |
| Caller-selected tenant route | Isolation bypass | Trusted identity-derived routing |

## What to measure

Track registry/DNS lookup latency and error rate; endpoint-set version and age; registration churn; readiness/liveness/startup changes; time from not-ready to traffic removal; stale-route calls; connections to draining endpoints; active-probe and passive-request disagreement; per-endpoint/zone latency, errors, saturation, and in-flight work; load-distribution skew; ejection count and duration; circuit state; retries and amplification; deadline exhaustion; cross-zone/region traffic; failover capacity; model/provider selection and fallback; ERP outcome-unknown age; and mTLS/certificate errors.

Alert on discovery cache age exceeding policy, endpoint-set collapse, correlated readiness loss, traffic continuing after drain, ejection near its cap, retry amplification, and routing to an ineligible region/version.

## Interview prompt

> Design discovery, health checking, and resilient routing for a multi-tenant AI operations platform deployed across three zones. It includes APIs, retrieval, model gateway, durable workflows, and ERP connectors. Endpoints autoscale and roll frequently; one zone can lose database connectivity; the discovery control plane can be unavailable; ERP writes may time out after succeeding; and model fallbacks must preserve tenant privacy and residency.

Cover discovery mechanisms, probe semantics, routing policies, draining, deadlines/retries, outlier detection, zone failure, control-plane outage, security, AI-provider routing, observability, and healthcare adaptation.

## Worked answer

**One-line conclusion:** Route logical services through versioned, capability-aware discovery; combine narrow readiness with real-request health; and bound every failover or retry by deadline, policy, idempotency, and authoritative workflow state.

“I would use platform-native or registry-backed endpoints with version, zone, protocol, and capability metadata. Instances pass startup checks before readiness, and planned shutdown marks not-ready before draining. Liveness remains local so a database outage does not restart the whole fleet.

Routers filter endpoints by trusted tenant region, service identity, API compatibility, and request class, then prefer the local zone using least-in-flight or weighted routing. Passive request outcomes supplement active probes. Outlier ejection uses rolling samples and a maximum ejected fraction; a dependency-wide circuit breaker handles correlated failure. Surviving zones have reserved headroom and admission control.

The caller propagates an end-to-end deadline. One layer owns bounded retries with jitter and a retry budget. Reads may retry or hedge when safe. ERP writes carry an operation ID; a timeout moves durable state to `outcome_unknown`, so routing to a different connector cannot cause an automatic duplicate. Reconciliation decides the next action.

During discovery failure, clients retain the last complete endpoint snapshot for a bounded interval and continue local passive health. They do not accept new regions, capabilities, or shard ownership without authoritative metadata. Long-lived connections have maximum age and graceful shutdown signals so endpoint removal actually drains traffic.

The model gateway routes only to evaluated providers compatible with privacy, residency, capability, quota, and deadline constraints. I would monitor endpoint-set age, readiness churn, drain convergence, per-zone saturation, ejections, circuit state, retries, deadline exhaustion, cross-region traffic, model fallbacks, and uncertain external outcomes.”

## Practice lab

Design routing behavior when:

- the API and retrieval fleets span three zones;
- one zone cannot reach PostgreSQL but still passes a shallow process probe;
- 20% of clients cache DNS beyond the intended TTL;
- a connector canary returns malformed responses only for large invoices;
- the discovery registry is unavailable for 15 minutes;
- one model provider returns 429 while another is healthy but not approved for PHI;
- an ERP write times out after possibly succeeding;
- long-lived streaming connections remain on draining instances;
- month-end traffic is 5× normal.

Specify startup/liveness/readiness checks, endpoint metadata, routing and locality policy, passive failure classification, circuit/ejection scope, drain timing, connection lifecycle, retry ownership, stale-discovery policy, model fallback eligibility, ERP reconciliation, and metrics.

Then state which behavior changes for hospital operations and why.

## References

- [Kubernetes — Configure liveness, readiness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes — Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Envoy — Outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier)
- [gRPC — Health checking](https://grpc.io/docs/guides/health-checking/)
- [AWS Builders’ Library — Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [Google SRE Book — Handling overload](https://sre.google/sre-book/handling-overload/)
