# Feature stores, point-in-time correctness, and training-serving consistency

A feature store manages reusable model inputs across training and online inference. Its hardest problems are not API convenience; they are time, lineage, identity, freshness, and producing the same feature meaning in every environment.

## Mental model

Treat a feature as a **versioned, time-indexed data product**, not a column copied into a model.

A feature value should answer:

- which entity does it describe?
- what event time does it represent?
- when did the platform learn it?
- which source and transformation produced it?
- which definition and version apply?
- how fresh is it allowed to be?
- can this tenant, model, and purpose use it?
- what happens when it is missing, late, corrected, or deleted?

A feature platform normally has five cooperating parts:

1. **feature registry** — definitions, owners, schemas, versions, keys, TTLs, lineage, and permissions;
2. **offline store** — historical feature values for training, evaluation, and batch scoring;
3. **online store** — low-latency latest values for real-time inference;
4. **materialization pipelines** — compute and synchronize values between sources and stores;
5. **retrieval SDK/service** — point-in-time historical joins and online feature vectors.

The core invariant is:

> For an inference decision at time (t), training and serving must use the same feature definition and only information that was legitimately available by (t).

## When to use it

A feature store is useful when:

- multiple models reuse the same business signals;
- online inference requires millisecond-to-low-second feature retrieval;
- training requires large historical point-in-time datasets;
- batch and streaming transformations must have consistent semantics;
- feature ownership, discoverability, lineage, or governance matters;
- teams repeatedly rebuild entity joins, freshness checks, and online caches;
- regulated or sensitive features need centralized access control;
- features must be backfilled, monitored, and deleted across consumers.

Do not build a feature platform merely because one batch model reads ten columns from one warehouse table. A versioned SQL view and reproducible training query may be enough.

A feature store is also not the source of truth for invoices, patients, approvals, or contracts. Those remain in authoritative operational systems. The feature store holds derived signals for model use.

## Architecture

A common flow is:

```text
Operational DBs / ERP / EHR / events / object storage
                    |
             batch + streaming transforms
                    |
        +-----------+------------+
        |                        |
 historical feature tables   online feature store
        |                        |
 point-in-time training      real-time retrieval
        +-----------+------------+
                    |
              model and decision log
```

The registry describes what may flow through that architecture. It does not guarantee correctness by itself.

Keep compute separate from storage where practical. The same logical transformation can be implemented for historical and streaming execution, but both paths need conformance tests because engines, windowing, late-data behavior, and numerical libraries can differ.

## Feature definition

A feature definition should include:

- stable feature and feature-view name;
- entity keys and tenant scope;
- value type, units, range, and null semantics;
- natural-language meaning;
- source contract and owner;
- transformation code and version;
- event-time column;
- availability or ingestion-time column;
- timestamp timezone and precision;
- window definition and inclusion boundaries;
- freshness target and TTL;
- default or missing-value policy;
- offline and online storage mapping;
- allowed purposes and sensitivity;
- lineage and downstream models;
- quality and drift checks;
- deprecation and compatibility policy.

For example:

```yaml
feature: vendor_late_invoice_rate_90d
entity: [tenant_id, vendor_id]
type: float
meaning: >
  Fraction of eligible invoices whose payment became more than 30 days late
  in the trailing 90-day window, excluding the current invoice.
event_time: invoice_due_at
availability_time: source_committed_at
window: "(prediction_time - 90d, prediction_time)"
freshness_slo: 30m
ttl: 2h
missing: null_with_indicator
owner: finance-data
version: 4
```

“Last 90 days” is incomplete unless the boundary, eligible population, current-event exclusion, lateness definition, corrections, and time zone are explicit.

## Entity keys and tenant isolation

Entities connect features to requests: vendor, project, equipment, patient, encounter, organization, or claim.

Use canonical typed keys. A vendor ID from one ERP is not automatically the same namespace as a vendor ID from another.

For multi-tenant systems, include trusted tenant identity in storage keys, retrieval authorization, caches, lineage, logs, and deletion workflows. Never accept `tenant_id` only from an untrusted model request.

Avoid entities with ambiguous lifetime. If equipment IDs can be reused, use an immutable equipment record ID plus validity interval rather than a display number alone.

Composite entities are sometimes required, such as `(tenant, project, subcontractor)`. Excessive high-cardinality combinations can explode storage and backfill cost, so model the actual decision boundary.

## Three clocks

Point-in-time correctness becomes clearer when distinguishing:

1. **event time** — when the business event happened;
2. **availability time** — when the system could legitimately use it;
3. **processing time** — when a pipeline computed or observed it.

Suppose an RFI was decided Monday, entered into Procore Wednesday, and processed Thursday. A Tuesday prediction must not see the decision merely because its business-effective date is Monday.

For training at prediction time (t), require both:

[
event_time le t
]

and, when reconstructing what the production system actually knew:

[
availability_time le t
]

The correct temporal rule depends on the model’s real serving contract. Write it down.

## Point-in-time joins

A normal join can leak future information:

```sql
SELECT invoice.*, vendor.current_late_rate
FROM invoice
JOIN vendor_features vendor USING (vendor_id);
```

This joins every historical invoice to today’s value.

A point-in-time join chooses the most recent feature row valid and available at or before each prediction timestamp:

```text
for each training example e:
    candidates = feature rows with matching entity
                 and feature_event_time <= e.prediction_time
                 and feature_available_time <= e.prediction_time
    choose the latest eligible row
```

Also enforce:

- tenant equality;
- feature TTL;
- source validity interval;
- exclusion of the target event;
- deterministic tie-breaking;
- no future corrections;
- known missing-value behavior.

For windowed features, compute the window relative to each example’s prediction time. Materializing today’s rolling aggregate and joining it backward is leakage.

## Label leakage

Leakage occurs whenever training uses information unavailable at inference, including:

- final payment status used to predict late payment;
- post-discharge notes used for admission-time prediction;
- a change-order approval created after an invoice decision;
- manual-review outcome included in features;
- a cancellation code entered after a no-show;
- a corrected source value backdated to before prediction;
- aggregate statistics containing the target row;
- random train/test splits that put later events in training.

Define a **prediction timestamp** and a **label-observation window** for every example.

Example: predict at invoice submission whether payment will exceed 30 days past due. Features stop at submission availability time. The label becomes observable only after the due date plus 30 days or a definitive payment event. Recent examples without mature labels must not be treated as negatives.

Use temporal validation splits and simulate production availability.

## Offline store

The offline store optimizes for:

- historical range scans;
- point-in-time joins;
- backfills;
- reproducible snapshots;
- training-set generation;
- batch scoring;
- lineage and deletion.

Partition carefully by event date and often tenant or entity hash, while avoiding tiny partitions. Store feature definition version and source lineage alongside values.

Training-set creation should output an immutable manifest containing:

- entity and prediction timestamps;
- selected feature versions;
- source/table snapshots;
- query or job version;
- point-in-time policy;
- label definition and maturity rule;
- row counts and quality results;
- content or snapshot digest.

A reproducible training dataset is a release artifact, not an ad hoc notebook result.

## Online store

The online store optimizes for:

- keyed low-latency reads;
- predictable tail latency;
- high availability;
- bounded payloads;
- freshness and TTL enforcement;
- atomic or version-consistent retrieval.

A stored record commonly includes value, feature version, event time, materialization time, expiry, and lineage pointer.

Decide atomicity. If a model requires ten mutually consistent features from one feature view, fetch a versioned bundle or snapshot rather than combining values from different materialization runs.

Online storage may be a key-value database, in-memory cache, or low-latency database. Choose from latency, scale, durability, regional, security, and operational needs—not brand familiarity.

The online store is derived state. Recovery should rebuild it from durable sources and checkpoints, although warm restoration may be needed to meet RTO.

## Materialization

Materialization moves computed feature values into online or offline stores.

Requirements:

- idempotent writes;
- version-aware upserts;
- monotonic or fenced ownership;
- checkpointed progress;
- late-event handling;
- backfill isolation;
- bounded retries;
- freshness monitoring;
- tenant fairness;
- deletion and correction propagation.

A stale worker must not overwrite a newer value. Compare source version, event/availability time, or fencing token at write.

Do not let a large historical backfill saturate the stream path that maintains online freshness. Use separate queues, quotas, and capacity.

A successful pipeline run is not enough; verify that expected values reached the serving store and can be retrieved.

## Batch and streaming consistency

Two implementations often exist:

- batch SQL/Spark for historical backfill;
- streaming code for live updates.

This creates training-serving skew.

Strategies:

### Shared transformation library

Reuse the same code.

Benefit: semantic consistency.  
Cost: the library must run efficiently in both environments and avoid hidden environment dependencies.

### Shared declarative definition

Compile one SQL/DSL/graph into batch and streaming plans.

Benefit: centralized meaning.  
Cost: the abstraction may not express every feature or engine behavior.

### Independent implementations with conformance tests

Benefit: each path is optimized.  
Cost: drift is likely without strong tests.

Whichever strategy is used, run golden-data tests comparing offline and online results for boundary conditions, late data, nulls, time zones, units, and corrections.

## Request-time features

Some features are only known at request time, such as current invoice amount or user-entered project.

Keep request features separate from stored features. Validate their schema and authorization, then combine them in a versioned feature service or model wrapper.

For derived request-time features, use deterministic transformations packaged with the model release. Do not recompute business meaning differently in a client, API gateway, and notebook.

Record the final feature vector—or a privacy-safe trace/digest with recoverable lineage—used for each consequential prediction.

## Missing and stale values

Missing can mean:

- entity does not exist;
- no historical events;
- source unavailable;
- pipeline delayed;
- value expired;
- authorization denied;
- schema rejected;
- deletion applied;
- lookup timed out.

Do not collapse these into zero.

Use explicit null/reason codes and, when appropriate, a missingness indicator. Defaults must reflect training behavior and business meaning.

Define per-feature freshness. A vendor’s 90-day rate may tolerate 30 minutes; current equipment telemetry may tolerate seconds. The feature service should surface event time, age, and status to the model wrapper.

Fallback options:

- use an evaluated default plus indicator;
- compute synchronously from an authoritative source within deadline;
- use a reduced model trained for the available subset;
- defer to batch or human review;
- reject/abstain for consequential decisions.

Never silently serve indefinitely stale values because the online store is reachable.

## Corrections, late data, and backfills

Late and corrected events create multiple truths:

- what happened;
- what the system knew then;
- what is known now.

For retrospective analytics, corrected event time may be appropriate. For faithful training simulation, preserve availability history so a backdated correction does not leak into old examples.

Use bitemporal or append-only records when this distinction matters:

- valid time: when the value applies in the business domain;
- system time: when the platform learned or stored it.

Backfills should specify feature version, source snapshot, time range, tenant scope, and conflict policy. Write to a new version or shadow table, validate, then promote. Do not mix partially backfilled versions.

## Feature versioning and compatibility

Create a new version when semantics change:

- window or boundary;
- source field;
- units or normalization;
- entity mapping;
- missing policy;
- availability rule;
- transformation;
- privacy or purpose restriction.

Schema-compatible does not mean semantically compatible.

Models bind to explicit feature versions. Deprecation requires:

1. identify downstream models and jobs;
2. stop new adoption;
3. migrate and evaluate consumers;
4. monitor old-version reads;
5. retire online values after the recovery window;
6. preserve required historical lineage or delete it under policy.

Aliases can aid migration, but production prediction logs must record resolved versions.

## Feature quality and monitoring

Before publication, check:

- schema and type;
- ranges and allowed categories;
- units;
- null and reason distribution;
- uniqueness and entity coverage;
- event/availability-time ordering;
- freshness;
- volume;
- duplicates;
- reconciliation with source totals;
- point-in-time test cases.

In production, monitor:

- offline/online parity;
- feature age and retrieval latency;
- missing/default rate by reason;
- distribution and drift by tenant/slice;
- outliers and impossible combinations;
- materialization lag;
- source-contract failures;
- version adoption;
- prediction/feature association;
- deletion completion.

Drift is a signal, not a diagnosis. It can reflect real business change, source breakage, population shift, a new tenant, or a bug. Connect alerts to source lineage and model outcomes.

## Security, privacy, and governance

Feature values can be more sensitive than raw fields because they infer behavior or risk.

Apply:

- tenant- and purpose-aware authorization;
- encryption and secret management;
- data minimization;
- field-level sensitivity tags;
- regional and residency controls;
- audit of retrieval and definition changes;
- retention and deletion propagation;
- approved-use restrictions;
- separation between production features and unrestricted exploration.

Do not allow a model or client to request arbitrary feature names. The deployed model release declares its eligible feature contract; the feature service authorizes the bundle.

For healthcare, a “risk score history” is PHI and may also create fairness concerns. Governance must cover derived features, not only source tables.

## Construction operations example

A platform predicts whether a newly submitted subcontractor invoice will require manual review or become more than 30 days late.

Prediction time: invoice accepted by the platform.

Features include:

- vendor late-payment rate over the prior 90 days;
- project budget variance as last known at submission;
- count of unresolved RFIs affecting the cost code;
- approved change-order value available at submission;
- invoice amount relative to remaining commitment;
- document-extraction confidence and validation failures;
- prior duplicate or dispute count;
- current request fields such as invoice amount.

Design:

1. use `(tenant_id, vendor_id)`, `(tenant_id, project_id)`, and commitment IDs as typed entities;
2. preserve event and availability time for ERP, Procore, and document events;
3. exclude the current invoice from vendor aggregates;
4. use point-in-time joins at invoice submission;
5. wait for mature labels rather than treating recent unpaid invoices as late;
6. materialize live aggregates to an online store with per-feature freshness;
7. combine request fields using the model’s versioned transformation bundle;
8. return missing/stale reason codes;
9. log resolved feature versions, values or protected lineage, and source ages;
10. route predictions to decision support—not automatic rejection—until business and fairness evidence supports more;
11. bind any downstream action to current authoritative records and approval;
12. propagate invoice/vendor corrections and deletion through offline, online, training, and prediction records.

A backdated change-order approval entered after submission must not appear in that invoice’s historical training row.

## Healthcare operations variation

A hospital predicts appointment no-show risk at scheduling time to choose an outreach workflow.

Features may include prior attendance, lead time, appointment type, communication preference, transportation indicators, and recent scheduling interactions.

Controls:

- define prediction time as the scheduling or reminder decision;
- exclude cancellation/no-show codes entered afterward;
- use availability time for delayed interface messages;
- distinguish “no prior visits” from source outage;
- enforce patient, organization, purpose-of-use, and region;
- minimize social or proxy features and evaluate fairness;
- bind feature versions to the released model;
- log the outreach decision and feature lineage;
- do not use the score to deny care;
- provide operational review and an approved fallback when features are stale.

A corrected encounter status may improve current records, but historical training reconstruction must preserve what was available when the earlier outreach decision occurred.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Central feature store | Reuse and governance | Platform complexity |
| Warehouse-only features | Simple batch workflow | Weak online latency |
| Online key-value store | Fast retrieval | Derived-state operations |
| Precomputed features | Low serving latency | Staleness and storage |
| Request-time computation | Fresh values | Latency and dependency risk |
| Shared batch/stream code | Semantic consistency | Runtime constraints |
| Independent paths | Performance flexibility | Training-serving skew |
| Point-in-time history | Honest evaluation | Storage and join complexity |
| Latest-value joins | Simple | Future leakage |
| Bitemporal history | Faithful corrections | Query complexity and cost |
| Strict TTL | Prevents stale decisions | Missing features during incidents |
| Default values | Availability | Hidden meaning and bias |
| Feature bundles | Consistent reads | Coarser updates and payloads |
| Fine-grained features | Reuse | More retrieval/version overhead |
| Global feature definitions | Standardization | Tenant semantics may differ |
| Tenant-specific variants | Correct local meaning | Version and evaluation explosion |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Join history to latest feature | Future leakage | Point-in-time join |
| Use event time only | Backdated data leaks | Availability-time constraint |
| Include target row in aggregate | Label leakage | Explicit exclusion boundary |
| Treat immature labels as negative | Biased training | Label maturity window |
| Random temporal split | Unrealistic evaluation | Time-based validation |
| Batch and stream differ | Training-serving skew | Shared definitions and parity tests |
| Feature meaning changes in place | Silent model regression | Semantic versioning |
| Tenant absent from key | Cross-tenant leak | Trusted tenant-scoped entities |
| Stale worker overwrites value | Freshness regression | Version/fencing checks |
| Backfill shares live capacity | Online freshness outage | Isolated queues and quotas |
| Online write succeeds partially | Mixed feature vector | Bundle/snapshot version |
| Missing becomes zero | Wrong business meaning | Reasoned nulls and indicators |
| TTL ignored during outage | Silent stale prediction | Freshness enforcement/fallback |
| Correction overwrites history | Cannot reconstruct decision | Append-only/bitemporal records |
| Source deletion misses online copy | Privacy violation | Lineage-driven deletion |
| Registry documents schema only | Semantic mismatch | Full feature contract |
| Model requests arbitrary features | Unauthorized inference | Release-bound allowlist |
| Drift alert lacks lineage | Slow diagnosis | Source-to-model trace |
| Online store treated as truth | Recovery/data corruption risk | Rebuild from durable sources |

## What to measure

Track point-in-time leakage tests; offline/online parity; feature freshness and materialization lag; retrieval p50/p95/p99; timeout and partial-read rate; missing/default reasons; entity coverage; source-to-feature reconciliation; batch/stream conformance; backfill progress and interference; feature-version adoption; drift and outliers by tenant/slice; model performance by feature availability; prediction reproducibility; deletion completion; cost per retrieved vector; and the percentage of consequential predictions traceable to exact feature and source versions.

A feature platform is successful when it prevents incorrect models and decisions—not when it has the largest catalog.

## Interview prompt

> Design a feature platform for a multi-tenant construction AI system. It trains and serves invoice-delay and manual-review models using ERP, project-management, document, and streaming events. It needs historical training, low-latency serving, late corrections, backfills, and tenant isolation. Adapt it to healthcare appointment operations with PHI.

Cover feature definitions, entities, event and availability time, point-in-time joins, offline and online stores, materialization, batch/stream consistency, request-time features, freshness, versioning, monitoring, security, deletion, and failure recovery.

## Worked answer

**One-line conclusion:** Define features as versioned time-indexed data products, train with availability-aware point-in-time joins, and serve the same semantics through freshness-enforced, tenant-scoped, lineage-rich online retrieval.

“I would keep operational systems authoritative and build a registry, offline historical store, online low-latency store, materialization pipelines, and retrieval service. Each feature contract defines entity and tenant keys, type, units, transformation, event time, availability time, window boundaries, TTL, missing policy, ownership, lineage, and allowed purpose.

Training examples have explicit prediction timestamps and mature label windows. The historical join selects only feature rows whose event and availability times are eligible, excludes the target event, enforces TTL, and uses deterministic tie-breaking. Time-based validation tests production reality.

Batch and streaming paths derive from a shared definition when possible and run golden parity tests for windows, time zones, late data, corrections, and nulls. Materialization is idempotent, checkpointed, version-aware, and isolated from backfills. The online store serves versioned feature bundles with values, event times, age, and missing reasons. Stale or partial vectors use an evaluated fallback, defer, or abstain.

Models bind to exact feature versions in the release registry. Prediction logs record the resolved versions and reconstructable lineage. Corrections preserve system-time history so old decisions can be reproduced; deletion follows lineage through offline, online, training, and decision records.

Tenant identity is part of keys and authorization, and the model release declares an allowed feature contract. I would monitor freshness, parity, missing reasons, drift, retrieval latency, source quality, version adoption, model outcomes, and deletion. Healthcare adds PHI purpose and region controls, fairness review, minimized features, and an explicit rule that operational risk scores cannot deny care.”

## Practice lab

Design point-in-time training and online serving for an invoice-delay model when:

- ERP invoices arrive through CDC;
- Procore change approvals can be entered days late and backdated;
- document-extraction confidence is corrected by reviewers;
- one tenant’s vendor IDs overlap another tenant’s;
- the 90-day vendor aggregate accidentally includes the current invoice;
- batch SQL uses UTC while streaming uses local time;
- the online store is 45 minutes stale;
- a year-long backfill competes with live updates;
- a feature definition changes its missing-value meaning;
- one vendor requests deletion;
- labels for recent invoices are not mature.

Specify entities, three clocks, feature contracts, point-in-time logic, label maturity, offline and online schemas, materialization conflict rules, freshness fallback, parity tests, version migration, deletion, and monitoring.

Then adapt the design to appointment no-show outreach using PHI, delayed HL7/FHIR messages, fairness-sensitive features, and a rule that the score cannot restrict care.

## References

- [Feast documentation](https://docs.feast.dev/)
- [Feast — Point-in-time joins](https://docs.feast.dev/getting-started/concepts/point-in-time-joins)
- [Uber Michelangelo: Machine Learning Platform](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
- [LinkedIn Feathr feature store](https://github.com/linkedin/feathr)
- [Rules of Machine Learning: Best Practices for ML Engineering](https://developers.google.com/machine-learning/guides/rules-of-ml)
- [Hidden Technical Debt in Machine Learning Systems](https://papers.nips.cc/paper/5656-hidden-technical-debt-in-machine-learning-systems)
- [Feature Store: Data System for Machine Learning](https://arxiv.org/abs/2103.14057)
