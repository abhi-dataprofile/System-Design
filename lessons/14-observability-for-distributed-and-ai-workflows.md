# 14 — Observability for distributed and AI workflows

Previous: [AI evaluations, guardrails, and release gates](13-ai-evaluations-guardrails-and-release-gates.md). Study time: 50–60 minutes including the exercise.

## Mental model

Monitoring answers questions you predicted. **Observability helps you explain an unfamiliar failure from the evidence a running system emits.**

Treat every user request or durable workflow as a causal story:

1. what the user intended;
2. which version and policy handled it;
3. where time and retries were spent;
4. which evidence, model, and tools influenced the result;
5. what state or side effect was committed;
6. whether the user received a correct, timely, authorized outcome.

Metrics show trends, logs explain discrete events, traces connect work across components, and profiles explain resource consumption. None replaces the others.

The key rule is:

> Observe the user outcome and every control boundary, while keeping sensitive data and unbounded identifiers out of telemetry indexes.

For an AI workflow, HTTP 200 is not success. The response can be slow, unsupported, based on stale evidence, blocked by a guardrail, or followed by a failed tool action. Technical health and product correctness must be measured separately.

## When to use each signal

| Signal | Best for | Example | Main risk |
| --- | --- | --- | --- |
| Metrics | Rates, distributions, alerts, capacity | p95 latency, queue age, grounded-answer rate | High-cardinality labels and misleading averages |
| Structured logs | Detailed events and audit clues | approval invalidated because version changed | Secrets/PHI, inconsistent fields, excessive volume |
| Distributed traces | Causal path and latency breakdown | API → retrieval → model → ERP adapter | Sampling can omit rare failures |
| Continuous profiles | CPU, memory, allocation, lock hotspots | tokenizer or parser consumes CPU | Overhead and access to sensitive symbols/data |
| Audit events | Security and business accountability | who approved which immutable plan | Confusing audit history with debug logs |
| Evaluation samples | Semantic quality | citation entails claim | Delayed signal and labeling cost |

Use metrics for detection, traces and logs for diagnosis, audit events for accountability, and evaluations for meaning.

## Start with service-level objectives

An SLO converts “reliable” into a measured promise over a window. Define a service-level indicator (SLI), target, population, and exclusions.

Examples:

- **interactive availability:** percentage of eligible read requests returning a usable response within 10 seconds;
- **workflow durability:** percentage of accepted invoice reviews that reach a correct terminal or explicitly reconciled state within 30 minutes;
- **freshness:** percentage of answers using the current published document version;
- **grounding:** percentage of eligible material claims supported by authorized citations;
- **action integrity:** percentage of write operations executed once, against the approved plan version, with a known outcome.

Do not exclude failures merely because a dependency caused them; users still experienced the failure. Exclusions should be narrow and written before the incident.

With a 99.9% target, the error budget is 0.1% of eligible events in the window. Spend it deliberately on releases and experiments. Fast burn means the current failure rate will exhaust the budget quickly; slow burn finds persistent degradation. Use multi-window burn-rate alerts so a short spike and a long leak both receive appropriate urgency.

## Define success before instrumentation

For each request class, write its terminal outcomes.

| Request | Success | Acceptable alternate | Failure |
| --- | --- | --- | --- |
| Cited question | Timely grounded answer | Explicit abstention/escalation | Unsupported or unauthorized answer |
| Document ingestion | Current version published | Quarantined with actionable reason | Silent loss or wrong version published |
| Invoice approval | Approved plan executed once | Outcome unknown and reconciling | Duplicate/wrong/unapproved action |
| Discharge summary | Authorized accurate draft | Clinician review requested | Cross-patient or unsupported clinical claim |

A timeout after sending an ERP request is neither success nor ordinary failure. Record `outcome_unknown`, show that state to the user, and measure time to reconciliation.

## Telemetry context and correlation

Propagate a small, trusted context through synchronous calls, queue messages, workflow checkpoints, model calls, and tools:

```text
trace_id
span_id / parent_span_id
request_id
workflow_id
operation_id
tenant_tier_or_hash_bucket
request_class
deployment_environment
service/version/region
model/prompt/index/tool/policy versions
```

Generate correlation IDs server-side. Do not trust a user-supplied tenant ID or trace field as authorization context.

For queues, inject trace context into message headers and create a consumer span linked to the producer. A retry may be a new delivery span linked to the same logical operation. Preserve the stable `workflow_id` and `operation_id` across retries, but create new attempt IDs.

For long-running workflows, one trace may become too large or exceed retention. Use one trace per stage or attempt plus span links and durable workflow events. The workflow timeline—not one enormous trace—is the source of truth.

## Design spans around boundaries

Create spans for meaningful work:

- request admission and authentication;
- authorization/policy decision;
- database and cache operations;
- queue publish, wait, and consume;
- retrieval query, reranking, and context assembly;
- model invocation;
- tool validation and execution;
- approval request and revalidation;
- external side effect and reconciliation.

Record bounded attributes such as operation name, result class, retry number, model version, token counts, retrieved-document count, cache hit, queue delay, and policy reason code.

Do not attach raw prompts, document text, access tokens, patient names, invoice descriptions, SQL, or unrestricted error bodies by default. Store sensitive diagnostic artifacts in a separate encrypted system with strict access, short retention, and audited retrieval; place only an opaque reference in telemetry.

## Metrics and cardinality

Metrics systems aggregate by label combinations. A label such as `workflow_id`, `user_id`, `invoice_id`, raw URL, or exception message creates unbounded time series and can overload the monitoring system.

Good labels are bounded dimensions used for action:

- service, environment, region;
- endpoint template or request class;
- success/error category;
- model or prompt version;
- tool name and result class;
- tenant tier, not tenant ID;
- controlled document or workflow type.

Keep IDs in traces or structured logs. Use exemplars to jump from a metric spike to a representative trace without turning trace IDs into labels.

Use histograms for latency, queue delay, token usage, document size, and cost. Report percentiles and bucket distributions, not averages alone. Separate queue time, compute time, dependency time, and streaming time-to-first-token from total latency.

## RED, USE, and workflow signals

For request-driven services, start with **RED**:

- rate;
- errors;
- duration.

For resources, use **USE**:

- utilization;
- saturation;
- errors.

Add workflow-specific signals:

- accepted, completed, failed, cancelled, and outcome-unknown counts;
- age of the oldest item in each state;
- state-transition duration;
- retry and duplicate-suppression rates;
- reconciliation backlog and age;
- approval wait, expiry, rejection, and invalidation;
- terminal outcomes by workflow version.

Queue depth alone is weak: 10 slow jobs may be worse than 10,000 tiny jobs. Queue age and projected drain time often better represent user impact.

## AI-specific observability

Do not reduce AI monitoring to tokens and latency. Capture four layers.

### Input and retrieval

- task, language, document type, length, and OCR-quality buckets;
- authorization-filter result;
- query/rewrite version;
- retrieved and reranked counts;
- evidence freshness and current-version match;
- empty retrieval and truncation rates.

### Model behavior

- model/provider, prompt, tool-schema, and routing versions;
- input/output tokens, time to first token, total duration, cost estimate;
- structured-output validity and repair attempts;
- finish/stop reason, refusal, abstention, and fallback;
- safety/guardrail result by bounded reason code.

### Tool and workflow behavior

- proposed versus executed tools;
- validation and authorization denials;
- retries, timeouts, idempotent duplicates, and uncertain outcomes;
- human review, correction, rejection, and escalation;
- budgets exhausted and no-progress stops.

### Quality

- citation coverage and entailment sample;
- groundedness, correctness, and task completion from evaluations;
- corrections and reversals by risk slice;
- model-judge agreement with experts;
- distribution drift in inputs, retrieval, and outcomes.

Quality signals may arrive hours or days later. Join them to the versioned workflow using opaque IDs in a controlled analytics store. Never alert on an uncalibrated model-judge score as if it were ground truth.

## Logs and event schemas

Emit structured events with stable schemas:

```json
{
  "event_name": "tool_execution_completed",
  "schema_version": 2,
  "timestamp": "...",
  "trace_id": "...",
  "workflow_id": "...",
  "operation_id": "...",
  "service_version": "...",
  "tool_name": "submit_invoice",
  "attempt": 2,
  "result_class": "outcome_unknown",
  "latency_ms": 4100,
  "policy_reason": "allowed",
  "artifact_ref": "restricted://..."
}
```

Separate fields that operators query from free text intended for humans. Normalize errors into stable categories; raw vendor messages vary and may contain secrets or personal data.

Avoid logging the same large payload at every hop. Define ownership for each event and use trace correlation.

## Sampling without hiding incidents

Head sampling decides before a trace finishes and is cheap, but it can discard rare slow or failed traces. Tail sampling decides after outcomes are known and can retain:

- all errors, authorization anomalies, and uncertain side effects;
- slow traces above request-class thresholds;
- canary and new-version traces;
- a representative sample of successes;
- selected high-risk workflow types.

Sampling does not replace metrics, which must count the full population. Publish and monitor effective sampling rates. During an incident, increase sampling selectively with a time limit; do not enable unrestricted sensitive-payload capture.

## Dashboards and alerts

Build dashboards in layers:

1. user journeys and SLO/error-budget burn;
2. workflow stages and business outcomes;
3. service RED and dependency health;
4. resource USE and capacity;
5. AI versions, quality, cost, and safety slices;
6. drill-down traces, logs, and runbook links.

Page on symptoms requiring urgent human action: fast SLO burn, cross-tenant security events, unauthorized writes, unreconciled high-value operations, or rapidly growing oldest-item age. Create tickets for slow burn, cost drift, weak evaluation slices, and noisy guardrails.

Every alert needs an owner, severity, user impact, evidence link, and first runbook action. If an alert fires repeatedly without action, fix or remove it. Alerting on every CPU spike creates fatigue and hides real incidents.

## Construction example

A project controller reports: “The assistant says the invoice was approved, but it is not in the ERP.”

The operator starts with the workflow ID from the user-visible status page:

1. the workflow timeline shows `approved → executing → outcome_unknown`;
2. the trace shows authorization and plan-version checks passed;
3. the tool span used operation ID `op_781`, timed out after sending the request, and did not blindly retry;
4. metrics show the ERP adapter timeout rate and reconciliation queue age increased after a connector release;
5. a deployment marker identifies the new adapter version;
6. the reconciliation worker queries the ERP by the same operation ID and finds the accepted invoice;
7. the workflow moves to `succeeded`, and an audit event records the final external identifier.

No raw invoice, bank details, or credential appears in the dashboard. Authorized responders can follow an artifact reference when payload inspection is necessary.

The incident review adds an SLO for reconciliation time, a canary test for the connector, and a regression case for “accepted but response lost.”

## Healthcare variation

A clinician sees a discharge draft citing an outdated medication list. The trace must identify the patient-scoped retrieval request, evidence versions, publication timestamps, model/prompt versions, and citation mapping without exposing PHI broadly in logs.

The workflow metric reports current-version correctness, while a quality review determines whether the outdated item materially changed the draft. Audit events show who viewed the restricted trace artifact. A safety alert fires if the system produces an unsupported medication instruction, not merely because model latency increased.

Clinical operations require stricter access, retention, redaction, and break-glass review. Debug telemetry is still regulated data when it contains patient context.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| More telemetry | Better diagnosis | Cost, noise, privacy exposure |
| Full tracing | Complete causal paths | Storage and runtime overhead |
| Head sampling | Cheap and immediate | Misses rare failures |
| Tail sampling | Retains interesting outcomes | Buffering and pipeline complexity |
| Raw prompt capture | Fast semantic debugging | Sensitive-data and injection risk |
| Opaque artifact reference | Safer controlled access | Slower investigation |
| High-cardinality labels | Flexible slicing | Monitoring cost and instability |
| Bounded labels plus logs | Stable metrics | More drill-down steps |
| Technical SLO only | Easy automation | Misses incorrect AI outcomes |
| Quality SLO | Tracks product value | Delayed labels and adjudication cost |
| Tenant-level dashboard | Customer diagnosis | Privacy and cardinality burden |
| Tier/bucket metrics | Efficient trends | Can hide one-tenant incident |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| HTTP 200 counted as success | Wrong answers look healthy | Outcome and quality SLIs |
| Average latency only | Tail pain hidden | Histograms and percentiles |
| User IDs as metric labels | Cardinality explosion | IDs in traces/logs only |
| Raw prompts in normal logs | Secret/PHI leakage | Minimize and use restricted artifacts |
| Trace context lost at queue | Broken causal chain | Propagate headers and span links |
| Retry reuses attempt ID | Attempts indistinguishable | Stable operation ID, unique attempt ID |
| Sampling drops failures | Incident invisible | Tail rules for errors and risk |
| One endless workflow trace | Oversized, incomplete trace | Stage traces plus durable timeline |
| Alert on infrastructure noise | Fatigue | Page on user/SLO impact |
| No deployment markers | Regression source unclear | Version attributes and release events |
| Model score treated as fact | False alert or reassurance | Calibrated reviews and uncertainty |
| Audit and debug logs mixed | Retention/access conflict | Separate purposes and controls |
| Dashboard has no runbook | Slow response | Owner, impact, and first action |
| Monitoring pipeline silently fails | False calm | Monitor telemetry freshness and loss |

## What to measure

At minimum:

- availability, latency, freshness, and action-integrity SLOs;
- error-budget consumption and burn rate;
- request rate, error class, latency distribution, and saturation;
- queue age, projected drain time, retry, DLQ, and duplicate suppression;
- workflow state age, completion, escalation, and reconciliation time;
- model latency, tokens, cost, structured-output repair, fallback, and refusal;
- retrieval emptiness, freshness, authorization filtering, and citation validity;
- tool authorization, timeout, uncertain outcome, and final reconciliation;
- quality and human corrections by language, document, risk, and version;
- telemetry drop rate, exporter backlog, sampling rate, and ingest delay.

## Interview prompt

> Design observability for a multi-tenant AI system that ingests construction documents, answers cited questions, reviews invoices, obtains approval, and submits them to an ERP. Workflows cross APIs, queues, model providers, retrieval systems, and external tools; they may last hours. Adapt the design for a healthcare discharge assistant.

Spend five minutes covering user outcomes, SLOs, metrics/logs/traces, context propagation, cardinality, privacy, AI-quality signals, sampling, dashboards, alerts, and an incident workflow.

## Worked answer

**One-line conclusion:** Trace each versioned workflow across control boundaries, alert on user-facing SLO burn and unsafe outcomes, and keep high-cardinality or sensitive evidence in access-controlled drill-down stores.

“I would define SLIs before choosing tools: timely grounded answers, current evidence, durable workflow completion, and exactly-once approved actions with known outcomes. Each request or workflow gets trusted trace, workflow, and operation IDs propagated through APIs, queue headers, checkpoints, model calls, and typed tools. Metrics cover RED, USE, queue age, workflow-state age, reconciliation, model cost and latency, retrieval freshness, corrections, and safety outcomes. Labels remain bounded; invoice, user, patient, and workflow IDs live in traces or structured logs. Spans record versions, counts, reason codes, and timings, never raw credentials or routine prompt/document contents. Sensitive artifacts use encrypted short-retention storage and audited references. Tail sampling retains all errors, slow requests, canaries, policy anomalies, and uncertain side effects while metrics count the whole population. Dashboards start with journeys and error-budget burn, then drill into stages, services, resources, and AI versions. Pages fire for rapid SLO burn, tenant-boundary violations, unauthorized writes, and aging uncertain outcomes. A stable operation ID connects an ERP timeout to reconciliation, while a unique attempt ID distinguishes retries. Incident reviews update runbooks, SLOs, and evaluation cases.”

## Practice lab

Design the telemetry and incident path for this failure:

- a controller approves invoice workflow `wf-91`;
- the queue delivers the execution message twice;
- the first ERP call times out after the request is sent;
- the second delivery is deduplicated;
- the reconciliation queue grows after a connector deployment;
- customer support reports that the UI still says “processing” after 40 minutes.

Define the SLI and SLO, IDs that remain stable or change, spans and structured events, bounded metric labels, sampling rule, page condition, dashboard drill-down, user-visible state, and final reconciliation evidence. Identify exactly what must **not** be logged.

## References

- [Google SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [NIST AI RMF Generative AI Profile](https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
