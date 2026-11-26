# Process mining, conformance checking, and operational bottleneck discovery

Dashboards summarize selected metrics. Distributed traces explain individual executions. Process mining reconstructs how business work actually flows across systems and people, then compares that observed behavior with the intended process.

It can reveal that estimates repeatedly return for missing scope, approvals bypass required reviews, invoices wait far longer between two departments than either team realizes, or AI-assisted cases close faster only because difficult cases are routed elsewhere.

## Mental model: build an evidence-backed map from event histories

Treat process mining as a governed analytical pipeline:

```text
source events + business objects + semantic mapping
  -> immutable normalized event log
  -> case/object correlation and lifecycle ordering
  -> process discovery, conformance, and performance analysis
  -> validated finding
  -> controlled workflow or policy change
  -> prospective outcome measurement
```

Each observed path is evidence of what occurred, not proof of what should occur or why it occurred.

Separate three major uses:

1. **Discovery:** infer common paths and variants from events.
2. **Conformance:** compare observed behavior with an approved model or rule set.
3. **Enhancement:** add time, cost, workload, quality, or prediction to an existing process model.

The goal is not the prettiest process diagram. It is a defensible operational finding tied back to source events.

## When to use it

Use process mining when:

- work crosses several applications, queues, teams, or organizations;
- stakeholders disagree about how the process operates;
- cycle time is high but the waiting stage is unclear;
- rework, bypasses, or manual workarounds may be hidden;
- policy compliance requires path-level evidence;
- a workflow redesign or AI assistant needs a baseline;
- cases contain repeated activities or many variants;
- operational outcomes must be compared by path, cohort, or version.

Use ordinary metrics when only a simple count or latency distribution is needed. Use tracing when diagnosing one technical request. Use qualitative discovery when event evidence is unavailable or misses important offline work.

Do not infer human intent, causal impact, or misconduct from sequence data alone.

## The minimum event contract

A classic process event needs:

| Field | Purpose |
|---|---|
| Event ID | Deduplicate and trace to source |
| Case ID | Correlate events for one process instance |
| Activity | Stable business meaning |
| Timestamp | Order and measure work |
| Lifecycle | Scheduled, started, suspended, resumed, completed, canceled |
| Source | System and record provenance |
| Event time and ingestion time | Separate occurrence from arrival |
| Actor/resource | Analyze handoffs and workload when permitted |
| Business object | Invoice, estimate, RFI, patient encounter, order |
| Tenant/project/facility | Scope and authorization |
| Outcome/status | Preserve the observed result |
| Schema and semantic version | Reproduce the interpretation |
| Attributes | Approved dimensions for slicing |

A timestamp without lifecycle semantics can confuse queue time with service time. A changed status row is not automatically a business event. Define activity meaning with domain owners.

Use globally unique event IDs or deterministic source keys. Never deduplicate merely by activity and timestamp: legitimate repeated work would disappear.

## Event semantics and provenance

For every mapped activity document:

- source table, API, audit log, or message;
- triggering condition;
- authoritative timestamp;
- timezone and precision;
- actor semantics;
- lifecycle transition;
- correction and deletion behavior;
- late-arrival expectation;
- whether absence means “did not occur” or “not observed”;
- semantic version and effective dates.

Preserve raw source events or immutable extracts. Build normalized events as reproducible derived data.

Do not overwrite event time when a record reaches the analytical pipeline. Maintain both:

- **event time:** when the business activity occurred;
- **ingestion time:** when the analytical system received it.

Backdated entries, offline synchronization, batch exports, and clock skew otherwise create false paths and negative durations.

## Case correlation is a modeling decision

The same events can tell different stories depending on the case definition.

Possible construction cases include:

- one bid opportunity;
- one estimate revision;
- one project;
- one invoice;
- one invoice line;
- one change order;
- one RFI.

A process instance should match the operational question. “How long does an estimate revision take?” needs a different case key from “How long from opportunity discovery to award?”

Document:

- start and end boundaries;
- business key and uniqueness;
- merge, split, and reopen semantics;
- version/revision handling;
- treatment of canceled and incomplete work;
- cross-tenant and cross-project collisions.

Test correlation using known examples. A wrong join can create impossible transitions that look like noncompliance.

## Object-centric process mining

Many processes do not fit one case ID. One invoice may cover several purchase orders; one change order can affect multiple invoices; several estimates may serve one opportunity.

Object-centric event logs connect an event to one or more typed objects:

```text
event: "cost review completed"
objects: opportunity-42, estimate-rev-3, project-17, reviewer-8
```

This avoids forcing many-to-many reality into a flattened case, but increases complexity. Define object types, relationships, qualifiers, and validity carefully.

Use classic case-centric logs when one primary object captures the question. Use object-centric modeling when convergence and divergence are themselves important.

## Process discovery

Discovery algorithms infer a model or directly-follows graph from event order. Before interpreting one:

- filter incomplete or corrupted cases;
- choose a time window and process version;
- distinguish start/complete lifecycle events;
- control variant noise;
- retain rare safety or compliance paths;
- separate tenants or policies when behavior is intentionally different;
- note concurrency rather than forcing arbitrary timestamp order.

A directly-follows edge means activity B was observed after A in a case. It does not establish that A caused B.

Useful discovery outputs include:

- dominant and rare variants;
- loops and rework;
- skipped stages;
- parallel branches;
- handoff patterns;
- paths associated with delay or correction;
- behavior by policy or workflow version.

Avoid “spaghetti” diagrams by starting from a question, not by loading every event into one view.

## Conformance checking

Conformance compares observed behavior with an expected model such as BPMN, Petri net, state machine, or declarative rules.

Evaluate more than one dimension:

- **fitness:** can the model reproduce observed traces?
- **precision:** does the model allow much behavior never observed?
- **generalization:** does it represent plausible unseen behavior without memorizing the log?
- **simplicity:** is the model understandable and maintainable?

Token replay and alignment-based methods can locate missing, extra, or out-of-order behavior. Interpretation still requires domain context.

Classify deviations:

| Deviation | Example | Meaning may be |
|---|---|---|
| Missing activity | No documented scope review | Bypass, logging gap, or valid exception |
| Extra activity | Repeated estimate correction | Rework or legitimate revision |
| Wrong order | Approval recorded before review | Clock issue, backdating, or policy breach |
| Unauthorized actor | Approval by wrong role | Identity mapping error or control failure |
| Deadline violation | Review after bid due date | Capacity, priority, or external delay |
| Attribute violation | High-value bid skips executive review | Policy version or configuration error |

Do not label a deviation “noncompliance” until event quality, policy version, and allowed exceptions are checked.

## Performance analysis: waiting is not working

Cycle time should be decomposed:

[
	ext{cycle time} = 	ext{queue time} + 	ext{service time} + 	ext{external waiting} + 	ext{rework} + 	ext{idle gaps}
]

For each stage measure:

- arrival-to-start wait;
- active service time;
- suspended or external wait;
- completion-to-next-stage handoff;
- rework count and time;
- percentiles and tail behavior;
- throughput and work in progress;
- abandonment and cancellation;
- outcomes and corrections.

If only completion timestamps exist, service time cannot be recovered accurately. Say so rather than treating inter-event time as employee work time.

Slice by relevant factors such as project type, estimate size, owner, source, policy, season, customer segment, model release, and evidence quality. Apply privacy and minimum-group rules.

A bottleneck is not simply the longest median duration. It may be a low-volume stage that blocks critical paths, a high-variability queue that creates tail delays, or a rework loop that multiplies downstream work.

## Data-quality tests

Before analysis, test:

- case IDs missing or reused;
- duplicate event IDs;
- impossible state transitions;
- timestamps outside source-record life;
- negative or extreme durations;
- timezone and daylight-saving errors;
- ingestion lag;
- actor identity changes;
- events missing during connector outage;
- semantic changes after software releases;
- unclosed cases near the observation window;
- suspiciously simultaneous batch timestamps;
- activity labels with uncontrolled cardinality;
- orphaned object relationships.

Report coverage: what fraction of known business records and time periods have usable event histories?

Use left and right censoring. A case already open at the window start lacks a known start; a case still open at the end lacks a known completion. Excluding these cases can make the process look faster than it is.

## Variant analysis without fooling yourself

A process can have thousands of unique paths. Normalize only semantics that are truly equivalent.

Group variants by:

- ordered activities;
- milestone sequence;
- loops and rework;
- role handoffs;
- exception type;
- workflow/policy version;
- outcome.

Keep frequency and business impact separate. A rare path may account for most disputed dollars or safety risk.

When comparing paths, account for case mix. High-value or complex estimates may both take longer and require more reviews. The review path is not necessarily causing the delay.

## Root cause versus association

Process mining finds patterns such as:

- cases with three handoffs close later;
- AI-assisted intake has fewer missing fields;
- one branch has more reopenings;
- winter projects wait longer for site verification.

These are associations. Confounding, selection, and policy routing may explain them.

For stronger claims:

- define a causal question;
- identify treatment, outcome, and assignment mechanism;
- control only appropriate pre-treatment variables;
- use temporal or randomized rollout where possible;
- measure prospective outcomes;
- check spillovers and delayed effects.

Do not remove a control merely because slow cases pass through it. The control may be detecting the risky cases it was designed to catch.

## AI-assisted process mining

AI can assist with:

- mapping source fields to candidate activity semantics;
- clustering free-text exception reasons;
- summarizing validated variants;
- generating natural-language queries;
- proposing hypotheses and missing instrumentation;
- explaining a conformance result from structured evidence.

Keep deterministic boundaries:

- the model does not invent events;
- every claim links to event IDs and query/version;
- sensitive text is minimized and authorized;
- activity mapping requires domain approval;
- generated SQL or transformations run in a constrained environment;
- narrative numbers are recomputed from structured results;
- protected attributes are governed;
- prompts cannot alter tenant scope.

LLMs are useful interfaces over a process model, not the source of operational truth.

## Architecture

A production platform can include:

1. **Source connectors** reading immutable audit logs, CDC, APIs, and approved exports.
2. **Raw event zone** preserving original payloads, source IDs, and ingestion metadata.
3. **Semantic mapping layer** producing versioned canonical activities.
4. **Identity and object resolver** connecting cases, revisions, and related objects.
5. **Quality service** publishing coverage and anomaly reports.
6. **Event-log warehouse** supporting temporal and tenant-scoped queries.
7. **Mining workers** calculating variants, models, alignments, and performance.
8. **Finding registry** storing query, dataset snapshot, algorithm, parameters, result, reviewer, and status.
9. **Review workspace** linking each conclusion to events and source records.
10. **Change/evaluation layer** connecting approved improvements to workflow versions and outcomes.

Use snapshot or watermark semantics so an analysis can be reproduced after late events arrive.

## Privacy, security, and workforce concerns

Event logs can expose employee behavior, patient activity, customer relationships, and operational strategy.

Apply:

- purpose limitation;
- least-privilege access;
- tenant and project scoping;
- pseudonymized actor analysis where names are unnecessary;
- minimum cohort sizes;
- retention limits;
- audit logs;
- legal and workforce review;
- restrictions on ranking individuals;
- separate access to PHI or protected attributes.

Process data often measures systems and queues more reliably than individual effort. A long task duration may mean waiting, multitasking, poor instrumentation, or an unusually hard case. Do not use it as a simplistic productivity score.

## Operationalizing a finding

A finding should contain:

- question and decision owner;
- source systems and coverage period;
- case/object definition;
- semantic-map and policy versions;
- filters, censoring, and exclusions;
- algorithm and parameters;
- uncertainty and limitations;
- event-backed evidence;
- business impact hypothesis;
- proposed intervention;
- guardrails and rollback;
- prospective measurement plan.

Move from observation to a controlled change:

```text
discover -> validate with operators -> design intervention
-> shadow or limited rollout -> measure outcome and harms
-> expand, revise, or roll back
```

Do not automatically change workflow policy because a mining job found a frequent path.

## Failure modes

| Failure mode | Defense |
|---|---|
| Wrong case key creates impossible paths | Domain-reviewed correlation and known-case tests |
| Batch timestamp is treated as event time | Preserve event and ingestion time |
| Inter-event gap is called work time | Capture lifecycle; label unresolved time honestly |
| Incomplete cases are excluded | Censoring-aware metrics and open-case cohorts |
| Rare compliance path is filtered as noise | Risk-aware retention and review |
| Policy versions are mixed | Version-specific conformance models |
| Parallel work is forced into false order | Lifecycle/concurrency semantics and partial ordering |
| Missing log is called a skipped step | Coverage checks and source reconciliation |
| Deviation is called misconduct | Validate semantics, exceptions, and context |
| Frequent path is assumed best | Compare outcome, risk, cost, and rework |
| Faster AI path reflects easier routing | Case-mix and prospective causal evaluation |
| Process diagram exposes sensitive staff data | Purpose-based access and pseudonymization |
| Late events silently change results | Versioned snapshot and watermark |
| LLM invents process facts | Event-linked structured evidence |
| Workflow changes but mappings do not | Semantic contracts and release tests |
| One customer dominates discovered behavior | Tenant/source weighting and cohort reporting |
| Analysis never changes operations | Finding owner, intervention, and evaluation plan |

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Case-centric log | Simple and mature tooling | Distorts many-to-many processes |
| Object-centric log | Represents shared objects | Higher modeling and query complexity |
| Fine-grained activities | Detailed diagnosis | Noise and unreadable models |
| Coarse activities | Clear process map | Hides rework and waiting |
| Aggressive variant filtering | Readable output | Removes rare, important behavior |
| Conformance to strict model | Clear control testing | Penalizes valid discretionary work |
| Rich actor dimensions | Handoff insight | Privacy and misuse risk |
| Near-real-time mining | Faster detection | Partial cases and higher cost |
| Periodic snapshots | Stable reproducibility | Slower response |
| Automated mapping | Faster onboarding | Semantic errors at scale |
| Manual domain mapping | Higher trust | Expensive and slower |
| AI narrative interface | Accessible findings | Hallucination and anchoring risk |

## Construction example: opportunity-to-estimate-to-bid

A general contractor wants to understand how opportunities move from discovery through go/no-go review, takeoff, pricing, subcontractor outreach, estimate review, bid submission, and award outcome.

Events come from a CRM, estimating platform, email gateway, document store, and bid portal. The team defines one opportunity as the primary case, links estimate revisions and bid submissions as objects, and versions activity mappings because the estimating platform changed its statuses midyear.

The event log distinguishes task start from completion. This prevents three days waiting for subcontractor quotes from being labeled as estimator work. It also records external-wait reasons, estimate revisions, scope clarifications, go/no-go decisions, and submission receipts.

Discovery shows a common loop:

```text
takeoff complete -> scope clarification -> takeoff revised -> pricing revised
```

Conformance finds some high-value opportunities reaching final pricing without the required executive review. Before declaring a control breach, the team checks effective policy dates and learns that several records were emergency-service opportunities under an approved exception; two others were mapping errors, and one is a genuine bypass.

Performance analysis shows the largest tail delay occurs between subcontractor outreach and quote coverage, concentrated in two trades and winter work. A second issue is late scope revision after pricing. The team validates both findings with estimators and tests earlier scope completeness checks plus structured quote reminders on a limited cohort.

Success is measured prospectively using bid timeliness, estimator rework, quote coverage, review compliance, overtime, win/loss outcomes, and estimator feedback. The system does not rank individual estimators from raw cycle time.

## Healthcare operations variation

For discharge operations, the process may connect admission, clinical clearance, pharmacy, equipment, transportation, authorization, caregiver readiness, and discharge.

Use an encounter or discharge episode as the governed case definition, and object-centric links when orders, medications, equipment, and authorizations are shared. Protect PHI, enforce minimum necessary access, version clinical policies, and distinguish operational waiting from clinical care.

Process mining may identify recurring operational waits, but it must not reinterpret clinical appropriateness. Validate deviations with clinicians, account for patient complexity, monitor access disparities, and test interventions prospectively.

## Interview prompt

**Design a process-mining platform for a construction company that wants to improve its opportunity-to-bid workflow across CRM, estimating software, email, documents, and bid portals. Events are incomplete, statuses changed after a software release, revisions share one opportunity, and leaders want bottlenecks, compliance deviations, and AI-assisted explanations. Adapt the design to healthcare discharge operations.**

## Worked answer

1. **Clarify the operational question.** Define the decision owner, outcome, time window, process boundary, and prohibited uses.
2. **Choose cases and objects.** Use opportunity as the primary case, link estimate revisions and submissions, and test many-to-many relationships.
3. **Create the event contract.** Preserve source/event IDs, activity, lifecycle, event and ingestion time, actor scope, object links, tenant, and semantic version.
4. **Build reproducible ingestion.** Store raw source evidence, normalize through versioned mappings, deduplicate by source identity, and watermark late events.
5. **Validate quality.** Measure coverage, duplicates, missing keys, clock errors, impossible order, connector gaps, censoring, and policy-version alignment.
6. **Discover behavior.** Calculate variants, loops, concurrency, handoffs, and outcome-linked paths without filtering rare risky cases away.
7. **Check conformance.** Compare traces with the effective approved model; distinguish logging gaps, valid exceptions, mapping faults, and genuine deviations.
8. **Find bottlenecks.** Separate queue, service, external-wait, rework, and handoff time; report tails and work in progress, not only averages.
9. **Bound AI.** Let models map candidates, cluster reasons, and explain structured findings, but require event citations and deterministic metric calculation.
10. **Govern access.** Scope by tenant/project, minimize actor identity, restrict workforce ranking, and protect sensitive text and attributes.
11. **Operationalize carefully.** Register the finding, validate it with operators, run a limited intervention, measure outcomes and harms, and roll back if needed.
12. **Adapt to healthcare.** Add PHI controls, clinical-policy versions, complexity adjustment, clinical validation, equity slices, and a boundary against clinical judgment.

**One-line conclusion:** Process mining is trustworthy only when every discovered path, deviation, and bottleneck can be reproduced from semantically versioned events—and operational changes are validated prospectively rather than inferred from a diagram.

## References

- [Process Mining Manifesto](https://www.tf-pm.org/resources/manifesto)
- [IEEE 1849-2016 — Extensible Event Stream Standard](https://standards.ieee.org/ieee/1849/6127/)
- [OCEL 2.0 specification](https://www.ocel-standard.org/)
- [PM4Py documentation](https://processintelligence.solutions/pm4py/)
- [OMG Business Process Model and Notation 2.0.2](https://www.omg.org/spec/BPMN/2.0.2/)
- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/)
- [HL7 FHIR Workflow module](https://hl7.org/fhir/workflow.html)
