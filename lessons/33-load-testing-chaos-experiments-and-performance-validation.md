# 33 — Load testing, chaos experiments, and performance validation

Previous: [Distributed transactions, sagas, compensation, and reconciliation](32-distributed-transactions-sagas-compensation-and-reconciliation.md). Study time: 55–65 minutes including the exercise.

## Mental model

Performance testing asks whether the system meets a measurable objective under a representative workload. Chaos engineering asks whether it preserves an explicit steady state when a controlled failure occurs.

> Test the workload shape and failure mode you expect, measure user and business outcomes—not only component utilization—and stop before an experiment becomes an uncontrolled incident.

A useful experiment has:

```text
hypothesis
+ representative workload
+ production-like constraints
+ observability
+ bounded fault
+ abort conditions
+ recovery verification
= actionable evidence
```

“Send many requests and watch CPU” is not a capacity test. “Kill random production instances” is not a responsible chaos program.

## When to use these tests

Use performance and resilience validation:

- before a major launch or customer onboarding;
- before month-end, seasonal, or event-driven peaks;
- after query, model, prompt, index, caching, or infrastructure changes;
- before and after horizontal-scaling changes;
- before regional failover or disaster-recovery claims;
- when setting SLOs, quotas, and capacity headroom;
- after an incident reveals an untested assumption;
- before raising automation authority for AI workflows.

Run the smallest test that answers the decision. A component benchmark can compare parsers; an end-to-end load test is needed to validate user latency, queues, dependencies, and side effects.

Never use real irreversible writes merely for realism. External ERP, payment, email, and clinical actions require sandboxes, simulators, isolated tenants, or dry-run adapters.

## Define success before generating load

Translate requirements into measurable objectives:

| Request/work class | Example objective |
| --- | --- |
| Interactive evidence read | Tail latency and success rate under target concurrency |
| Document upload | Durable acceptance latency and bounded backlog |
| OCR/embedding | Pages or tokens processed per minute and drain time |
| AI response | Time to first token, completion latency, quality, and cost |
| Approval execution | Correct terminal outcome and no duplicate effect |
| ERP reconciliation | Unknown-outcome age and recovery throughput |
| Human review | Queue age and reviewer capacity |
| Failover | Recovery time, lost/duplicated work, and degraded behavior |

Do not invent numerical thresholds. Derive them from user deadlines, contracts, baseline measurements, dependency quotas, and business risk.

Record:

- tested build, configuration, model, prompt, policy, and dataset;
- infrastructure size and autoscaling bounds;
- workload mix and key distribution;
- cache state;
- dependency behavior;
- test duration;
- pass/fail gates;
- known differences from production.

## Test taxonomy

### Baseline

Run one or a few representative operations to establish correctness and per-request resource cost. Baselines reveal instrumentation errors before scale hides them.

### Load test

Exercise expected steady traffic and normal bursts. Validate objectives, resource use, and headroom.

### Stress test

Increase load until objectives fail. Find the bottleneck and behavior beyond capacity. The goal is not merely the maximum request rate; it is the failure boundary and whether overload is controlled.

### Spike test

Apply a rapid demand increase such as morning login or month-end upload. Validate queue bounds, autoscaling delay, admission, cache protection, and recovery.

### Soak test

Run a realistic workload long enough to expose leaks, fragmentation, connection exhaustion, growing queues, stale credentials, log-volume problems, and scheduled-job interference.

### Scalability test

Change resources or replicas and measure whether useful throughput scales, where contention begins, and whether cost per successful outcome improves.

### Failover test

Remove a zone, replica, provider, or control-plane component and measure recovery, correctness, capacity, and failback.

### Chaos experiment

Inject a bounded fault to test a specific resilience hypothesis: latency, packet loss, dependency error, process termination, clock skew in a sandbox, disk pressure, expired credential, stale discovery, or queue redelivery.

These are complementary. A service that survives instance termination may still collapse under a 5× spike.

## Workload modeling

A realistic workload model includes:

- operation mix;
- arrival pattern;
- concurrency;
- payload and document size distribution;
- tenant/key skew;
- read/write ratio;
- cache hit/miss state;
- session behavior;
- think time;
- model input/output token distribution;
- retries and cancellations;
- background work and scheduled jobs;
- external dependency quotas and latency;
- data volume and index cardinality.

Use distributions, not averages. A mean 10-page document does not represent a mix of nine 1-page invoices and one 91-page drawing set.

Model dominant tenants and hot projects explicitly. Uniform random tenant IDs hide noisy-neighbor and hot-partition failures.

## Open versus closed workload models

A **closed model** keeps a fixed number of virtual users. A user sends a new request only after the previous one completes. When the service slows, generated request rate falls.

An **open model** sends arrivals at an independent rate. When the service slows, concurrency and queues grow as real demand may.

Closed tests are useful for interactive session behavior. Open tests expose overload and queueing more honestly. State which model you use.

A generator that waits for each response can make an overloaded system appear stable because it quietly reduces offered load.

## Coordinated omission

Coordinated omission occurs when the test generator pauses arrivals while the system is slow and therefore fails to record requests that users would have attempted.

Example: a service stalls for 10 seconds. A closed-loop client with one user records one 10-second request. A real arrival stream of 100 requests/s would create roughly 1,000 affected requests and a backlog.

Use constant/independent arrival scheduling where appropriate, record intended start time, and report queue delay plus service time. Verify that the load generator itself is not saturated.

## Capacity relationships

Little’s Law provides a reasonableness check:

[
L = lambda W
]

where (L) is average in-flight work, (lambda) is completed throughput, and (W) is average time in system.

If throughput is 40 requests/s and average time in system is 0.5 seconds, expect about 20 concurrent requests. Large disagreement suggests measurement boundaries, queueing, drops, or non-steady conditions need investigation.

Utilization near 100% often causes nonlinear queue growth. Capacity planning should use tail latency, recovery behavior, and headroom—not only average utilization.

## Latency measurement

Measure the full path:

```text
client scheduling
+ network
+ admission/queue
+ service
+ dependency
+ serialization/stream
= user-observed latency
```

Report a distribution: p50, p90, p95, p99, maximum with context, and histogram. Percentiles must state the population, interval, and success/error treatment.

Avoid averaging percentiles across instances or time windows. Aggregate raw histogram-compatible data or calculate from the combined distribution.

For asynchronous work measure:

- submit-to-durable-acceptance;
- queue wait;
- execution;
- completion-to-publication;
- end-to-end age;
- backlog drain time.

A fast `202 Accepted` is not a fast completed workflow.

## Throughput and correctness

Count:

- offered work;
- admitted work;
- completed successful work;
- rejected/deferred/degraded work;
- retries and attempts;
- duplicate effects;
- timed-out but later completed work;
- corrupted or missing outputs.

Throughput means useful, correct outcomes per unit time. Retrying the same request three times should not triple throughput.

For a test involving external writes, verify business invariants and operation ledgers after the run. HTTP success rate alone cannot prove absence of duplicates or lost events.

## Production-like environment

A test environment should resemble the production dimensions relevant to the hypothesis:

- database size, indexes, and query plans;
- network paths and zones;
- instance types and resource limits;
- connection pools;
- autoscaling minimum/maximum and cooldown;
- queue partitions and retention;
- cache capacity and cold/warm state;
- model deployment, batching, and quotas;
- service mesh/proxy;
- security checks;
- representative tenant skew;
- downstream simulators.

Perfect duplication is rarely affordable. Document deviations and avoid extrapolating across unknown bottlenecks.

Use synthetic or de-identified data with realistic shape. Preserve size, cardinality, skew, language, document quality, and relationships without copying sensitive production records.

## Test isolation and safety

Before a significant test define:

- owner and incident commander;
- environment and tenant isolation;
- approved time window;
- maximum offered load;
- downstream call budget;
- irreversible-action blocks;
- dashboards and alerts;
- abort thresholds;
- kill switch;
- rollback/reset;
- data cleanup;
- stakeholder notification.

Production tests need smaller blast radius, established observability, and immediate abort. Start with development, then staging, then a narrow canary or shadow experiment.

Protect other tenants with quotas and dedicated test identities. Mark test traffic in traces without allowing the marker to bypass authorization or safety logic.

## Chaos engineering workflow

A responsible chaos experiment:

1. define the steady-state indicator;
2. form one falsifiable hypothesis;
3. verify baseline health;
4. choose the smallest fault and blast radius;
5. specify duration and abort conditions;
6. notify/authorize operators;
7. inject the fault;
8. observe user, business, and system outcomes;
9. stop or abort;
10. verify recovery and cleanup;
11. document evidence and gaps;
12. create owned remediation;
13. rerun after the fix.

Example hypothesis:

> If one retrieval-zone dependency adds 800 ms latency for five minutes, interactive requests remain within the degraded-response objective, no cross-region privacy fallback occurs, and backlog drains within the recovery objective.

The fault, success criteria, and safety boundary are all testable.

## Fault catalog

Useful faults include:

- process/container termination;
- zone capacity loss;
- dependency latency or errors;
- connection reset;
- DNS/registry staleness;
- queue redelivery or consumer pause;
- database replica lag;
- cache loss;
- disk/full or I/O throttling in nonproduction;
- expired credential/certificate;
- provider quota exhaustion;
- model endpoint failure;
- worker pause beyond lease;
- delayed/out-of-order events;
- corrupted nonproduction payload;
- control-plane unavailability.

Inject one fault first. Compound faults are valuable only after the individual mechanisms are understood and the blast radius is justified.

Do not inject destructive data corruption into production merely to prove backups. Restore verified copies in an isolated environment.

## Abort conditions

Abort on signals such as:

- tenant isolation or authorization failure;
- irreversible external side effect;
- data corruption;
- error budget consumption beyond approved limit;
- critical queue age;
- outcome-unknown actions above bound;
- surviving-zone saturation;
- monitoring loss;
- inability to stop the injector;
- unexpected blast-radius expansion.

An abort is a successful safety mechanism, not a failed experiment. Record why it fired and whether cleanup worked.

## Overload and recovery

Test behavior beyond the knee:

- early admission rejection;
- bounded queues;
- priority/reserved capacity;
- retry budgets;
- circuit breakers;
- fair scheduling;
- degraded modes;
- autoscaling lag;
- recovery after offered load drops.

A system that survives overload but takes hours to drain stale work may still fail the business objective.

Measure hysteresis: the path back to health may differ from the path into overload because queues, caches, connection pools, and retries remain elevated.

## Failure injection and external effects

For consequential workflows, inject failures at exact boundaries:

- before local commit;
- after commit but before outbox publish;
- after publish before consumer acknowledgment;
- before external send;
- after external acceptance before response;
- during compensation;
- during reconciliation.

Then verify:

- idempotency;
- no lost workflow;
- explicit outcome-unknown state;
- no blind retry;
- correct compensation/forward recovery;
- operator evidence;
- eventual reconciliation.

A generic “kill a pod” test may miss the dangerous millisecond between ERP acceptance and response.

## AI performance validation

AI systems require more than request latency.

Measure:

- time to first token;
- inter-token latency;
- end-to-end completion;
- input/output tokens;
- tokens per second;
- batch formation and queue delay;
- GPU/CPU memory and utilization;
- cache hit rate;
- retrieval latency and candidate count;
- tool-call count and latency;
- fallback/degraded-mode rate;
- cost per correct outcome;
- quality and safety under load.

Performance changes can alter quality. Under pressure the system may truncate context, reduce retrieval candidates, route to a smaller model, time out tools, or return partial output. Re-run evaluation slices during load.

Keep risk-critical evaluation gates constant. Do not claim success because latency improved if citation correctness or tool safety regressed.

For streaming, distinguish time to first visible token from time to validated completion. A fast partial answer followed by failure may be worse than a slightly slower complete one.

## Load-generator correctness

Validate the test system:

- enough CPU/network/socket capacity;
- synchronized measurement clocks where required;
- no hidden client connection bottleneck;
- realistic connection reuse;
- correct think time;
- request payload variation;
- generated IDs and idempotency behavior;
- accurate error classification;
- preserved intended arrival times;
- independent metrics from the system under test.

Distribute generators when necessary. If one generator maxes out at 2,000 requests/s, a flat result does not prove the service capacity is 2,000 requests/s.

## Baselines and regression gates

Store results with versioned configuration. Compare:

- same workload and dataset;
- confidence intervals or repeated-run variance;
- tail latency;
- throughput at objective;
- error/rejection/degradation mix;
- resource and cost per successful outcome;
- correctness and AI quality;
- recovery time.

Avoid a single universal percentage gate when natural variance differs by scenario. Investigate statistically and operationally meaningful changes.

A regression test should fail on a real user or cost objective, not only an unstable microbenchmark.

## Construction operations example

A platform expects 5× month-end invoice traffic while a 100,000-page backfill runs. OCR, retrieval, model inference, ERP reconciliation, and human review share constrained capacity.

Test plan:

1. replay a synthetic mix preserving page, token, tenant, project, and invoice-size distributions;
2. include one tenant with 40% of pages and one hot project;
3. use an open arrival model for uploads and interactive reads;
4. keep the backfill active throughout a soak period;
5. test cold and warm caches;
6. cap external ERP calls through a simulator matching latency, quota, timeout, and idempotency behavior;
7. inject an outcome-unknown response after simulated ERP acceptance;
8. remove one worker zone and delay model responses;
9. verify reserved capacity for interactive evidence and reconciliation;
10. measure durable acceptance, queue age, drain time, tail latency, quality, cost, fairness, duplicates, and unknown outcomes;
11. abort on isolation failure, unbounded queue growth, or unsafe external attempt;
12. confirm the backlog drains and all operations reconcile after load returns to normal.

Pass criteria come from the agreed month-end user deadlines, quota limits, and business recovery objectives—not invented numbers.

## Healthcare operations variation

A healthcare operations test uses synthetic patients and approved test tenants. It preserves realistic resource cardinality, FHIR payload shape, scheduling bursts, document size, language, and interface latency without exposing PHI.

Experiments verify that overload never bypasses authorization, PHI never routes to an unapproved region/model, and emergency operational capacity remains available. A model fallback must pass the same privacy and task-specific evaluation gates.

Chaos tests run first outside production and use bounded test identities. Clinical source systems use sandboxes or simulators. Abort immediately on isolation, authorization, or unexpected real-patient impact. Recovery evidence includes freshness, delayed-message reconciliation, and manual-path readiness—not merely service uptime.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Component benchmark | Fast bottleneck isolation | Misses system interactions |
| End-to-end test | User-realistic | Expensive and harder to diagnose |
| Closed workload | Models user sessions | Hides overload through slowdown |
| Open workload | Reveals queueing | Can create large backlog |
| Production-like staging | Safer | May differ at scale |
| Production canary test | Highest fidelity | Real blast radius |
| Synthetic data | Privacy and repeatability | Shape may be unrealistic |
| Traffic replay | Realistic mix | Privacy and side-effect risks |
| Warm-cache test | Common steady state | Hides cold-start behavior |
| Cold-cache test | Validates recovery/startup | Can be pessimistic |
| Single fault | Clear diagnosis | Misses interactions |
| Compound fault | Tests real incident complexity | Harder attribution and safety |
| Automatic abort | Limits harm | May stop before learning |
| Long soak | Finds slow leaks | Time and cost |
| AI fallback under load | Availability | Quality/privacy regression risk |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Test uses average payload | Tail bottleneck missed | Representative distributions |
| Uniform tenants | Hot-key/noisy-neighbor hidden | Preserve skew |
| Closed loop only | Offered load drops during slowdown | Open arrivals where appropriate |
| Coordinated omission | Tail impact understated | Record intended schedule |
| Generator saturates | False service capacity limit | Monitor/distribute generators |
| p99 averaged across hosts | Invalid percentile | Merge distributions/histograms |
| Only HTTP latency measured | Queue/business delay hidden | End-to-end state timings |
| Retry attempts count as throughput | Inflated capacity | Count correct unique outcomes |
| Warm cache only | Failover storm missed | Test cold and warm |
| Staging is tiny | Invalid production extrapolation | Document/scale relevant dimensions |
| Test data lacks cardinality | Index/query plans differ | Shape-realistic synthetic data |
| Chaos has no abort | Experiment becomes incident | Predefined automated stop |
| Kill pod only | Critical boundary untested | Target specific failure points |
| Fault injection persists | Recovery cannot complete | Cleanup verification |
| External sandbox differs | False confidence | State limitation and reconcile |
| Load test sends real ERP/email | Business harm | Simulator/dry-run isolation |
| AI latency improves via truncation | Quality silently falls | Evaluate quality under load |
| Autoscaling masks cold-start errors | Intermittent failures | Inspect new-instance readiness |
| Queue survives but never drains | Long business outage | Measure recovery/drain time |
| Test marker bypasses policy | Unrealistic security result | Same authorization path |

## What to measure

Measure offered/admitted/completed/rejected/degraded work; unique correct outcomes; latency distributions; queue wait, age, and drain time; concurrency; saturation; autoscaling timing; per-tenant fairness; cache behavior; database/query metrics; retries and amplification; circuit state; cost per outcome; error-budget use; failover and recovery time; data loss/duplicates; outcome-unknown and reconciliation age; compensation; and operator intervention.

For AI add TTFT, inter-token latency, tokens/s, truncation, retrieval recall/citation correctness, tool success, safety violations, fallback route, and evaluated quality by risk slice.

For experiments record blast radius, abort signal, fault start/stop, detection time, mitigation time, recovery verification, hypothesis result, and remediation owner.

## Interview prompt

> Design a performance and chaos-validation program for a multi-tenant construction AI platform before month-end. Traffic may reach 5× normal, one tenant produces 40% of pages, a 100,000-page backfill is running, model and ERP quotas are constrained, and an availability zone may fail. ERP calls can succeed before timing out, and AI fallback can change answer quality.

Cover workload modeling, open/closed arrivals, coordinated omission, metrics, environment fidelity, overload, safe fault injection, external effects, AI quality, abort conditions, recovery, and healthcare adaptation.

## Worked answer

**One-line conclusion:** Validate representative work and bounded failures against explicit user, correctness, recovery, and AI-quality objectives—then treat overload recovery and experiment safety as first-class results.

“I would begin with request classes and business objectives: interactive evidence, durable uploads, OCR/embedding completion, model TTFT and quality, approval execution, ERP reconciliation, and review-queue age. The synthetic dataset preserves page/token distributions, tenant skew, hot projects, index cardinality, and late/duplicate events.

I would use open arrivals for bursty uploads and API demand, closed sessions for user journeys, and record intended start times to avoid coordinated omission. Tests progress from baseline to expected load, spike, stress, and soak. I measure unique correct outcomes, latency histograms, queue wait and drain time, saturation, fairness, retries, cost, duplicates, and unknown external outcomes.

External writes use an ERP simulator with realistic quotas, latency, idempotency, and lookup. Fault injection targets the boundary after simulated acceptance but before response, proving the workflow enters outcome unknown and reconciles rather than blindly retrying. Other experiments remove one zone, delay model calls, pause consumers, and clear caches.

Each chaos test has a steady-state hypothesis, small blast radius, abort conditions, kill switch, and recovery verification. I abort on isolation failure, unbounded critical queues, unsafe effects, monitoring loss, or excessive surviving-zone saturation.

AI validation includes TTFT, completion latency, tokens, cost, retrieval/citation quality, tool correctness, and safety by risk slice. Fallback is allowed only to evaluated, privacy-compatible models. Success requires backlog drain, invariant verification, reconciliation, and acceptable quality—not merely restored CPU and HTTP success.”

## Practice lab

Create a validation plan for:

- 5× month-end arrivals;
- one tenant produces 40% of document pages;
- a 100,000-page backfill;
- cold cache and autoscaling from minimum capacity;
- one-zone loss;
- database replica lag;
- model latency plus quota exhaustion;
- ERP acceptance followed by timeout;
- queue redelivery;
- 60-minute steady load followed by recovery.

Specify workload distributions, open/closed models, objectives, telemetry, fault injection points, abort conditions, external simulators, AI evaluation slices, pass/fail evidence, backlog-drain validation, and remediation workflow.

Then adapt the experiment to healthcare without using PHI or risking real clinical actions.

## References

- [Google SRE Book — Testing for Reliability](https://sre.google/sre-book/testing-reliability/)
- [Google SRE Workbook — Non-Abstract Large System Design](https://sre.google/workbook/non-abstract-design/)
- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [AWS Builders’ Library — Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/)
- [Gil Tene — How NOT to Measure Latency](https://www.youtube.com/watch?v=lJ8ydIuPFeU)
- [OpenTelemetry — Metrics data model](https://opentelemetry.io/docs/specs/otel/metrics/data-model/)
