# Semantic layers, metrics contracts, and trustworthy natural-language analytics

A warehouse can contain correct tables while dashboards still disagree. One team counts invoices, another counts invoice lines, a third excludes voided records after aggregation, and an AI assistant selects whichever column name sounds plausible.

A semantic layer gives business concepts stable, governed meaning above physical storage. A metrics contract defines exactly how a measure is calculated, filtered, joined, secured, versioned, and tested. Natural-language analytics should compile questions through this layer rather than inventing SQL over raw schemas.

## Mental model: compile business questions into authorized metric plans

Treat analytics as a typed compilation pipeline:

```text
user identity + business question + authorized semantic catalog
  -> resolve metric, dimensions, filters, time, and grain
  -> produce a structured query plan
  -> validate joins, policy, cost, and ambiguity
  -> compile to SQL or engine request
  -> execute on a governed snapshot
  -> validate and present result with lineage and caveats
```

Keep four layers separate:

1. **Physical layer:** source tables, files, columns, partitions, and indexes.
2. **Model layer:** cleaned entities, facts, dimensions, relationships, and history.
3. **Semantic layer:** business names, measures, dimensions, metrics, policies, and valid joins.
4. **Experience layer:** dashboards, notebooks, APIs, alerts, and natural-language assistants.

The same metric definition should serve every experience. The LLM may interpret language; it does not define revenue, project completion, utilization, or margin.

## When to use it

Use a semantic and metrics layer when:

- several dashboards or teams calculate the same concept;
- physical schemas differ across ERP, CRM, project, or clinical systems;
- self-service analytics must remain governed;
- joins can duplicate facts;
- time zones, revisions, snapshots, or slowly changing dimensions matter;
- row, column, tenant, project, or facility security must be consistent;
- natural-language questions need a bounded vocabulary;
- metric changes require impact analysis and controlled rollout;
- APIs or operational workflows consume analytical measures.

A small team with one stable report may begin with a tested view. Add a semantic service when reuse, policy, scale, or natural-language access justifies it.

Do not use a semantic layer to hide poor source quality. It should expose freshness, lineage, and limitations.

## Start with grain

Every fact model and metric must state its grain in one sentence.

Examples:

- one row per invoice;
- one row per invoice line revision;
- one row per project per accounting month;
- one row per employee per time-entry interval;
- one row per dumpster service event;
- one row per patient encounter per day.

Before defining a measure, ask:

- What does one row represent?
- Which key makes it unique?
- Can the record be revised, voided, reopened, or backdated?
- Is this event, transaction, periodic snapshot, or accumulating snapshot data?
- Which timestamp determines inclusion?
- Can one business object appear in more than one row?

If grain is ambiguous, totals will be ambiguous.

## Measures, dimensions, metrics, and entities

Use precise terms:

| Concept | Meaning | Example |
|---|---|---|
| Entity | Identifiable business object | Project, vendor, invoice, dumpster |
| Dimension | Attribute used to group or filter | Project type, region, status |
| Measure | Aggregatable numeric expression | Sum of approved amount |
| Metric | Governed business calculation with context | Approved invoice dollars this month |
| Derived metric | Calculation from other metrics | Gross margin percentage |
| Cohort | Entity group defined by an event/time rule | Projects started in Q1 |
| Snapshot | State as observed at a declared time | Month-end committed cost |

A metric is more than a SQL expression. It needs ownership, purpose, grain, time semantics, filters, units, security, freshness, lineage, and tests.

## Metrics contract

A production metric contract should declare:

- stable metric ID and human name;
- description and business question;
- owner and approver;
- semantic version and effective dates;
- source facts and lineage;
- base grain and entity keys;
- measure expression and aggregation type;
- required filters and exclusions;
- dimensions allowed for grouping;
- valid join paths and cardinality;
- primary and alternate time dimensions;
- timezone and business calendar;
- currency and unit conversion;
- null, unknown, void, and late-arrival behavior;
- snapshot or revision semantics;
- row/column policy and sensitivity;
- freshness expectation;
- completeness and quality thresholds;
- comparable historical range;
- deprecation and migration path;
- example queries and expected results.

Treat a breaking change in business meaning like an API breaking change. A renamed column is not the only breaking change; changing “approved” to include conditionally approved invoices is semantic breakage.

## Aggregation semantics

Measures behave differently:

- **additive:** may be summed across all relevant dimensions, such as invoice amount at a nonduplicated grain;
- **semi-additive:** may be summed across some dimensions but not time, such as month-end balance;
- **non-additive:** ratios, distinct counts, medians, and percentiles require recomputation;
- **holistic:** exact median or percentile cannot be derived from independently aggregated medians;
- **approximate:** sketches may support scalable distinct count or quantiles with stated error.

Do not average averages without compatible denominators. Store or derive numerator and denominator, then calculate the ratio at the requested grain.

For gross margin:

[
	ext{gross margin %} =
rac{sum 	ext{recognized revenue} - sum 	ext{recognized cost}}
{sum 	ext{recognized revenue}}
]

It is not the unweighted average of project margin percentages.

## Join safety and fanout

A one-to-many join can multiply facts.

Suppose one invoice has five lines and three approval events. Joining invoice lines to approval events by invoice can produce fifteen rows. Summing line amount then triples the invoice.

The semantic graph should declare:

- primary and foreign entities;
- one-to-one, one-to-many, and many-to-many relationships;
- join keys and tenant scope;
- temporal validity;
- allowed traversal direction;
- bridge tables;
- fanout-safe measures;
- required pre-aggregation.

Reject ambiguous or unsafe plans. Do not rely on `DISTINCT` as a universal repair; two legitimate rows may have the same amount.

Use tests with adversarial fixtures: multiple lines, repeated approvals, shared vendors, revised records, overlapping IDs across tenants, and missing dimensions.

## Time is part of metric meaning

A question such as “revenue in October” is incomplete until time semantics are chosen.

Possible times include:

- created;
- submitted;
- approved;
- posted;
- service performed;
- recognized;
- paid;
- corrected;
- observed in the warehouse.

Define:

- event or snapshot time;
- timezone;
- fiscal and business calendar;
- inclusive/exclusive boundaries;
- late and backdated events;
- daylight-saving behavior;
- “as of” versus “currently known” interpretation;
- current versus historical dimension attributes.

For bitemporal data, distinguish when something was valid in the business from when the system learned it. A report reproduced “as known on October 31” may differ from a later corrected report about October.

## Slowly changing dimensions and history

A vendor, project manager, cost code, facility, or region may change.

Choose deliberately:

- **Type 1:** overwrite history; old facts use the current attribute.
- **Type 2:** preserve effective-dated dimension versions.
- **Snapshot:** materialize state at defined observation points.
- **Event history:** reconstruct state from changes.

A project transferred between managers should not silently rewrite historical workload unless that is the intended metric. Define whether attribution uses the manager at event time, report time, or current ownership.

## Metric layers and serving architecture

A production design can include:

1. **Source contracts** for ERP, CRM, project, estimating, and operational systems.
2. **Ingestion and history** using snapshots, CDC, or event logs.
3. **Transformation layer** creating tested facts and dimensions.
4. **Semantic catalog** defining entities, relationships, measures, metrics, dimensions, and policies.
5. **Planner/compiler** resolving valid metric queries.
6. **Execution engine** querying warehouse, lakehouse, OLAP store, or federated sources.
7. **Cache/aggregate manager** keyed by metric version, policy scope, filters, and snapshot.
8. **Lineage and quality service** connecting result to source and tests.
9. **Experience APIs** serving dashboards, notebooks, alerts, and AI.
10. **Evaluation and audit store** retaining plans, SQL, snapshots, results, and feedback.

Keep the semantic definition source-controlled. Promote changes through test and review environments. Generate documentation from the same definitions used by execution.

## Authorization belongs in planning and execution

A natural-language assistant must not rely on prompt instructions such as “only show this project.”

Derive authorization from trusted identity and enforce it in:

- catalog discovery;
- metric and dimension availability;
- row filters;
- join paths;
- compiled query;
- warehouse session or secured views;
- result cache;
- export and drill-through.

A user may be allowed to see project totals but not employee pay, patient details, vendor banking data, or small sensitive cohorts.

Cache keys must include authorization and semantic versions. Never share cached results across tenants merely because the question text matches.

Use minimum-group suppression or other approved disclosure controls where aggregate results can reveal individuals.

## Natural-language analytics pipeline

A reliable assistant should:

1. authenticate the user and resolve scope;
2. retrieve only authorized metric definitions and examples;
3. parse the question into a structured intent;
4. identify metric, dimensions, filters, time range, comparison, and sort;
5. surface unresolved ambiguity;
6. build a semantic query plan;
7. validate grain, join cardinality, policy, freshness, and cost;
8. compile using deterministic code;
9. execute read-only with limits;
10. verify result shape and invariants;
11. render a concise answer with definition, time basis, filters, freshness, and lineage;
12. retain the question, plan, versions, SQL, result digest, and user feedback.

Prefer a constrained intermediate representation:

```json
{
  "metric": "approved_invoice_amount",
  "group_by": ["project"],
  "time_dimension": "approval_date",
  "time_range": ["2026-10-01", "2026-11-01"],
  "filters": [{"field": "invoice_status", "op": "in", "value": ["approved"]}],
  "comparison": "previous_period"
}
```

The model proposes this plan. Deterministic services verify that each field and combination is allowed.

## Ambiguity and clarification

Common ambiguities include:

- “revenue” billed, recognized, collected, or contracted;
- “hours” scheduled, clocked, estimated, or approved;
- “completed project” substantial completion, financial close, or customer sign-off;
- “this month” calendar, fiscal, rolling 30 days, or month to date;
- “best estimator” speed, accuracy, win rate, margin, or volume.

If one interpretation is dominant and low risk, answer with the definition shown. If interpretations materially change the result, request or present a bounded choice in an interactive setting. For non-interactive reports, state the selected assumption and provide alternatives where important.

Never silently choose a sensitive or consequential metric because its name resembles the prompt.

## SQL and execution safety

Even read-only analytics can be harmful through data exposure, resource exhaustion, or misleading results.

Controls include:

- allowlisted semantic plans rather than arbitrary SQL;
- parser and logical-plan validation;
- read-only credentials;
- tenant and row policy enforcement;
- query timeout, scan, row, memory, and concurrency limits;
- result-size and export limits;
- blocked functions and external network access;
- cost estimation;
- cancellation;
- isolated execution;
- parameter binding;
- audit logs.

If generated SQL is supported for expert workflows, validate the abstract syntax tree and query plan, not only text patterns.

Do not let query results trigger business actions directly. An analytical finding can create a proposal or case through a separately authorized workflow.

## Freshness, quality, and truthful answers

Every result should expose:

- data-as-of time;
- latest complete period;
- source coverage;
- metric version;
- policy scope;
- known incidents or degraded sources;
- estimated or approximate status;
- suppressed groups;
- relevant caveats.

Decide per metric whether stale data should:

- fail closed;
- return with a warning;
- use the latest complete snapshot;
- fall back to an alternate source.

A precise number from incomplete data is not a trustworthy answer.

Quality tests can cover:

- uniqueness and referential integrity;
- accepted statuses and units;
- reconciliation to authoritative totals;
- freshness and volume;
- null and unknown rates;
- tenant leakage;
- join fanout;
- late-arrival changes;
- invariant checks;
- metric fixtures.

## Evaluation

Evaluate the full system, not only text-to-SQL syntax.

Build a versioned test suite with:

- canonical business questions;
- ambiguous questions;
- paraphrases and spelling errors;
- multi-turn refinements;
- unsupported metrics;
- unsafe joins;
- unauthorized requests;
- prompt injection inside catalog text;
- stale or partial data;
- time-zone and fiscal-calendar boundaries;
- empty results;
- large scans;
- adversarial and cross-tenant identifiers.

Score:

- metric selection;
- dimension and filter selection;
- time interpretation;
- semantic-plan validity;
- execution success;
- numerical correctness;
- authorization;
- clarification/abstention;
- citation and definition accuracy;
- latency and cost;
- answer usefulness.

Use trusted expected plans and results, not only model judges. Compare releases by question slice and investigate regressions.

## Change management and lineage

A metric change can affect dashboards, alerts, APIs, models, experiments, and executive reports.

Maintain:

- dependency graph from source to metric to consumer;
- owners and subscribers;
- contract tests;
- parallel old/new versions;
- backfill or recomputation plan;
- effective date;
- historical comparability note;
- migration examples;
- deprecation window;
- rollback.

When a meaning changes, decide whether to restate history or preserve the old definition. Never splice incompatible definitions into one trend line without marking the boundary.

## Failure modes

| Failure mode | Defense |
|---|---|
| Same name, different business meanings | Stable metric IDs, descriptions, owners, versions |
| Invoice lines multiply after approval join | Cardinality graph and fanout tests |
| Average of project margins misstates portfolio margin | Recompute from additive numerator and denominator |
| Current manager rewrites historical attribution | Explicit effective-time dimension semantics |
| October report changes without explanation | Bitemporal/as-of policy and restatement marker |
| AI invents a column or metric | Authorized semantic catalog and typed plan |
| Assistant chooses billed instead of recognized revenue | Ambiguity handling and visible definition |
| Prompt text bypasses row policy | Identity-derived enforcement in planner and warehouse |
| Cached result leaks another tenant | Policy-scoped cache key |
| Stale ERP feed returns confident total | Freshness gate and latest-complete-period label |
| SQL is syntactically valid but semantically wrong | Expected-plan/result evaluation |
| DISTINCT hides a many-to-many join bug | Grain-aware aggregation and adversarial fixtures |
| Metric change silently breaks dashboards | Dependency graph and versioned rollout |
| Small cohort exposes sensitive information | Suppression and disclosure controls |
| Query overloads warehouse | Cost, scan, timeout, concurrency, and result limits |
| Model judge approves wrong number | Deterministic numeric oracle and reconciled fixtures |
| Analytics answer triggers an action | Separate approval-bound operational workflow |

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Central semantic service | Consistent definitions | Availability and platform dependency |
| Definitions compiled into each tool | Lower runtime dependency | Drift across consumers |
| Flexible raw SQL | Maximum exploration | Semantic inconsistency and data risk |
| Constrained metric queries | Safer and reproducible | Less expressive |
| Pre-aggregations | Low latency and cost | Freshness and invalidation complexity |
| Live warehouse queries | Current, flexible data | Variable latency and cost |
| Strict freshness failure | Avoids misleading results | Reduced availability |
| Warn on stale data | Maintains access | Users may ignore warning |
| One canonical metric | Simplicity | May hide legitimate contexts |
| Versioned contextual metrics | Honest meaning | More discovery and migration work |
| AI clarification | Better intent resolution | Extra interaction and latency |
| Default interpretation | Fast answers | Silent semantic error |
| Rich lineage in every answer | Trust and debugging | UI complexity |
| Fine-grained authorization | Stronger protection | Planning and caching complexity |

## Construction example: trustworthy project-performance analytics

A general contractor wants a natural-language assistant for questions such as:

- “How many estimator hours did we spend on bids submitted last quarter?”
- “Which active projects are over budget?”
- “What is our bid win rate by project type?”
- “How long do invoices wait for approval?”
- “Which dumpster customers have containers sitting longer than the agreed period?”

The source systems contain opportunities, estimate revisions, time entries, projects, commitments, change orders, invoices, service events, and payments.

The team defines:

- **estimator hours:** approved time-entry duration attributed to an estimate revision, not the elapsed time between estimate events;
- **bid win rate:** awarded opportunities divided by decided opportunities, excluding open and canceled opportunities under a versioned policy;
- **project budget variance:** approved forecast-at-completion minus current approved budget, both from the same accounting snapshot;
- **invoice approval time:** submission to final approval, with external-wait and reopened intervals reported separately;
- **container dwell:** service-ready placement to pickup/exchange, using contract-specific free days.

Each definition declares grain, time basis, revision behavior, dimensions, join path, source freshness, and authorization.

A question about “hours to complete an estimate” is ambiguous. The assistant distinguishes labor hours from elapsed cycle time and shows the selected definition. It cannot infer employee productivity from elapsed time.

The project-margin metric recomputes total recognized revenue and cost before division. Estimate revisions are not blindly joined to every time entry and opportunity status. Current and historical project-manager attribution are separate dimensions.

Results include the data-as-of time, metric definitions, filters, fiscal period, source coverage, and lineage. If ERP CDC is delayed, financial metrics return the latest complete snapshot with an explicit warning; operational service-event metrics may remain current.

## Healthcare operations variation

A healthcare operations layer may define bed occupancy, length of stay, discharge delay, appointment utilization, denial rate, and staffing coverage.

Clinical and operational meanings require approved owners. Distinguish census snapshots from encounters, scheduled from completed appointments, and operational delay from clinical necessity. Enforce minimum necessary PHI access, small-cell suppression, facility and role policies, approved time zones, and audit logs.

A natural-language assistant can answer authorized operational questions, but it must not invent clinical interpretations, expose patient-level details through drill-down, or convert an analytical correlation into treatment advice.

## Interview prompt

**Design a semantic metrics platform and natural-language analytics assistant for a construction company using CRM, estimating, timekeeping, project-management, ERP, and dumpster-dispatch systems. Dashboards currently disagree, joins cause duplication, historical ownership changes, source feeds become stale, and users have different project permissions. Adapt the design to healthcare operations.**

## Worked answer

1. **Inventory questions and owners.** Identify high-value decisions, existing definitions, authoritative sources, consumers, and security boundaries.
2. **Model grain and entities.** Define facts, dimensions, revisions, snapshots, event time, business keys, and tenant-scoped relationships.
3. **Create metric contracts.** Declare formula, aggregation, filters, time, units, join paths, freshness, policy, quality, versions, and examples.
4. **Build a semantic graph.** Expose authorized entities, measures, dimensions, and cardinality-safe traversal; reject fanout and ambiguity.
5. **Compile through a typed plan.** Let the LLM propose metric intent, then deterministically validate and compile it rather than accepting free-form SQL.
6. **Enforce authorization twice.** Filter catalog discovery and execution with trusted identity; scope caches, exports, and drill-through.
7. **Protect execution.** Use read-only credentials, AST/plan validation, limits, timeouts, cancellation, isolation, and cost estimation.
8. **Return truthful context.** Show definitions, filters, time basis, snapshot, freshness, lineage, approximation, and caveats with every answer.
9. **Test semantics.** Use reconciled fixtures for fanout, revisions, ratios, time zones, stale feeds, unauthorized requests, and ambiguous prompts.
10. **Version changes.** Track dependencies, run old/new metrics in parallel, mark history breaks, migrate consumers, and retain rollback.
11. **Observe outcomes.** Monitor query correctness, clarification, abstention, policy denials, stale answers, warehouse cost, latency, and user corrections.
12. **Adapt to healthcare.** Add approved clinical/operational ownership, PHI minimization, small-group protection, encounter/snapshot semantics, and a boundary against clinical advice.

**One-line conclusion:** Trustworthy natural-language analytics requires the model to interpret questions while a versioned semantic layer—not generated SQL or column names—owns business meaning, join safety, authorization, and numerical truth.

## References

- [dbt Semantic Layer documentation](https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl)
- [MetricFlow documentation](https://docs.getdbt.com/docs/build/about-metricflow)
- [Google Cloud Looker modeling documentation](https://cloud.google.com/looker/docs/what-is-lookml)
- [Cube semantic layer documentation](https://cube.dev/docs/product/introduction)
- [OpenLineage specification](https://openlineage.io/docs/spec/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [OWASP Top 10 for Large Language Model Applications](https://genai.owasp.org/llm-top-10/)
