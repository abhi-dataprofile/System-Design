# 25 — End-to-end capstone: multi-tenant AI operations assistant

Previous: [FDE discovery, integration architecture, and phased delivery](24-fde-discovery-integration-architecture-and-phased-delivery.md). Study time: 65–80 minutes including the exercise.

## Mental model

A strong system design is a chain of justified boundaries, not a list of technologies.

```text
outcome and risk
  → requirements and estimates
  → authoritative data and trust boundaries
  → APIs and durable workflows
  → retrieval, reasoning, and human decisions
  → idempotent external actions
  → observability, recovery, and improvement
```

For an AI operations assistant, split the system into two planes:

- the **evidence plane** ingests, authorizes, versions, indexes, and retrieves facts;
- the **action plane** proposes, reviews, executes, and reconciles consequential changes.

A model may help interpret evidence or propose a plan. It must not become the source of identity, authorization, money, clinical facts, workflow state, or external outcomes.

> Keep authoritative state deterministic, make AI outputs evidence-bound proposals, and turn every consequential action into a durable, approval-bound, idempotent workflow.

## When to use this architecture

Use this design when a platform must:

- serve multiple organizations with strict tenant isolation;
- ingest documents, APIs, events, email, or database changes;
- answer questions with source citations and current authorization;
- run long-lived workflows that survive retries and deployment;
- propose actions against ERP, project, scheduling, or clinical systems;
- require human approval for high-risk decisions;
- support evaluation, audit, rollback, deletion, and disaster recovery;
- expose tenant customization without customer-specific code forks.

Use a simpler architecture when the task is a low-risk internal search tool, a bounded offline analysis, or a deterministic single-system workflow. Do not introduce agents, queues, vector search, or stream processing unless requirements justify them.

## 1. Clarify requirements before drawing

Start with actors and top journeys:

1. authorized user asks a cited question;
2. source document or record changes;
3. system proposes an operational action;
4. reviewer approves, rejects, or requests information;
5. external system confirms, rejects, or leaves outcome unknown;
6. operator investigates failure or incident;
7. tenant requests export, retention, or deletion.

Define functional requirements:

- tenant/project/facility-scoped search;
- citations to immutable source versions;
- structured extraction and validation;
- configurable review policy;
- approval-bound actions;
- durable status and audit;
- connector reconciliation;
- feedback and evaluation;
- admin configuration with safe bounds.

Define nonfunctional requirements:

- availability and latency by journey;
- freshness by source;
- peak workload and tenant skew;
- recovery point and recovery time;
- privacy, residency, retention, and audit;
- cost per successful outcome;
- graceful degradation;
- onboarding and support ownership.

Ask what must never happen:

- cross-tenant retrieval;
- action from stale or unauthorized approval;
- duplicate payment or clinical action;
- unsupported medical authority;
- silent deletion failure;
- fabricated successful external outcome.

## 2. Back-of-the-envelope sizing

Use transparent variables instead of invented precision.

Assume:

- (T) active tenants;
- (U) daily active users per tenant;
- (Q) questions per user per day;
- (D) new document pages per tenant per day;
- (A) proposed actions per tenant per day;
- (B) peak-to-average traffic multiplier;
- (C) average chunks per page;
- (E) embedding bytes per chunk including metadata/index overhead.

Then:

[
	ext{daily queries} = T 	imes U 	imes Q
]

[
	ext{daily chunks} = T 	imes D 	imes C
]

[
	ext{annual vector footprint} approx 365 	imes T 	imes D 	imes C 	imes E
]

[
	ext{peak query rate} approx rac{T 	imes U 	imes Q}{	ext{active seconds/day}} 	imes B
]

Estimate separately:

- API request concurrency;
- upload bytes and object growth;
- OCR/parser CPU;
- embedding tokens and rate limits;
- model input/output tokens;
- vector and lexical index size;
- workflow/queue throughput;
- reviewer minutes and backlog;
- connector quotas;
- audit/log volume.

Capacity must be workload-aware. A 500-page drawing set is not one unit equivalent to a small invoice.

## 3. High-level architecture

```text
Users / Admins
      |
Edge gateway + identity
      |
Application APIs ---------------- Configuration / policy control plane
      |                                      |
      +---- Query service ---- Retrieval ----+
      |          |                |
      |          |          lexical + vector indexes
      |          |                |
      |          +---- model gateway / answer composer
      |
      +---- Workflow service ---- review queues
                   |
             durable commands
                   |
          connector workers ---- ERP / project / scheduling systems

Sources ---- ingestion API / CDC / webhooks
                   |
          object quarantine + metadata DB
                   |
       parse / extract / chunk / embed
                   |
       versioned publication to indexes

Across all components:
tenant authorization, secrets/KMS, audit, metrics/logs/traces,
evaluation, reconciliation, retention/deletion, backup/DR
```

The metadata database stores authoritative platform state: tenants, resource versions, workflow states, approvals, operation IDs, lineage, configuration references, and audit pointers.

Object storage holds immutable source files and derived artifacts. Search indexes are rebuildable projections, not the source of truth.

## 4. Tenant and identity boundary

Authenticate through an enterprise identity provider or approved login. The server resolves:

- user/service identity;
- tenant membership;
- project/facility/resource relationships;
- roles and attributes;
- purpose and policy context.

Never accept a model-generated or browser-supplied tenant ID as authoritative.

Enforce tenant scope at multiple layers:

- API authorization;
- database keys and row policies where useful;
- object paths and signed URL scope;
- queue/event envelopes;
- search filters before ranking;
- cache keys;
- workflow and connector credentials;
- logs, traces, evaluation data, and replay tools.

Use workload identity between services. Keep tenant connector credentials in a secrets manager, ideally isolated by tenant and environment.

## 5. Data model and source of truth

Core records:

- `Tenant`, `UserMembership`, `ProjectOrFacility`;
- `SourceConnection`;
- `Resource` and immutable `ResourceVersion`;
- `IngestionRun` and `ArtifactLineage`;
- `Chunk` and `IndexPublication`;
- `Conversation` and bounded `Message`;
- `WorkflowRun`, `Task`, and `Checkpoint`;
- `Proposal`, `ActionPlan`, `ApprovalDecision`;
- `Operation` and `ExternalOutcome`;
- `PolicyBundle`, `ConfigSnapshot`, `EvaluationBundle`;
- `RetentionState`, `DeletionOperation`, and audit events.

Every tenant-owned table has tenant identity in its primary or unique keys. Every mutable resource has a version used for optimistic concurrency.

Do not copy vendor semantics blindly. Owned adapters map sources into a versioned canonical domain model while preserving original references and lineage.

## 6. Safe ingestion path

For direct upload:

1. API authenticates and authorizes tenant/project scope.
2. Server creates a pending resource version and bounded upload grant.
3. Client uploads directly to quarantine object storage.
4. Completion event carries tenant, resource/version, size, hash, and media type.
5. Worker validates checksum, type, size, malware policy, and archive limits.
6. Parser runs in an isolated, resource-bounded environment.
7. Extraction records page/field provenance and confidence.
8. Chunker creates deterministic IDs with parser/chunker versions.
9. Embeddings write to a versioned index namespace.
10. Publication transaction marks the complete version active.
11. Old chunks become inactive and later purge under retention policy.

For APIs, webhooks, and CDC:

- verify signatures and source versions;
- durably accept before slow work;
- deduplicate stable event IDs;
- map through the owned adapter;
- handle out-of-order changes with resource versions;
- reconcile periodically against the source;
- quarantine ambiguous or invalid records.

Never publish a partially indexed document as current.

## 7. Retrieval and answer flow

Query flow:

1. authenticate user and resolve tenant/resource scope;
2. classify request as read-only answer, workflow request, or unsupported;
3. normalize query without changing business meaning;
4. select retrieval sources and current index versions;
5. apply authorization filters before candidate ranking;
6. combine lexical, semantic, metadata, and structured lookup;
7. rerank a bounded candidate set;
8. construct context with citations, source versions, and token budget;
9. call an approved model through a gateway;
10. validate structured output and citation references;
11. return answer, uncertainty, and missing-evidence state;
12. log lineage and quality metadata without copying unnecessary sensitive content.

Hybrid retrieval helps when exact invoice numbers, vendor names, codes, and semantic questions coexist. Structured queries should answer exact balances or statuses; do not force all data through embeddings.

If evidence is insufficient or conflicting, the correct output is “cannot establish from current authorized sources,” not a confident completion.

## 8. Model gateway and routing

The model gateway owns:

- approved providers/models and region;
- workload-specific routing;
- token and cost budgets;
- timeouts and retry policy;
- prompt and tool schema versions;
- structured-output validation;
- sensitive-data rules;
- rate limits and tenant quotas;
- fallback;
- request/response lineage;
- circuit breaking.

Route by capability, risk, latency, and cost. A small model may classify or extract bounded fields; a larger model may synthesize a cited answer. Deterministic validation remains outside the model.

A model timeout can retry only when the request has no external side effect. Tool actions occur through durable workflows, never as an untracked model call.

## 9. Agent versus workflow

Use a deterministic workflow when steps and transitions are known. Use an agent only for bounded planning or interpretation where multiple evidence/tool paths are genuinely useful.

A safe controller loop:

1. load durable state and budgets;
2. construct authorized context;
3. ask model for a typed proposal;
4. validate tool and arguments;
5. evaluate policy;
6. execute read-only tool or create an approval task;
7. append result/checkpoint;
8. continue within step, token, time, and cost limits;
9. stop on completion, insufficient evidence, or escalation.

The model cannot directly mutate workflow state. Tools are typed, scoped, versioned, observable, and independently authorized.

Memory is not one bucket:

- conversation state has bounded retention;
- workflow state is authoritative and durable;
- user preferences are explicit configuration;
- retrieved enterprise facts remain source-linked;
- model scratch reasoning is not durable business truth.

## 10. Approval-bound action flow

For a proposed ERP or operational write:

1. create normalized `ActionPlan`;
2. bind tenant, target resource, expected source version, arguments, evidence manifest, policy/config/tool versions, expiry, and operation ID;
3. hash the canonical plan;
4. route by risk to qualified reviewers;
5. store immutable decisions with identity and reason;
6. before execution, reevaluate current authorization, policy, expiry, and resource versions;
7. mark stale and return to review if material inputs changed;
8. execute through a connector using the stable operation ID/idempotency key;
9. record confirmed success, rejection, or outcome unknown;
10. reconcile uncertain outcomes before any retry.

The UI shows the exact effect, evidence, uncertainty, and changes since proposal creation.

## 11. Connector boundary

Each connector owns vendor-specific:

- authentication and secret rotation;
- schemas and semantic mapping;
- pagination and incremental cursor;
- webhooks/polling/CDC;
- quotas and bounded concurrency;
- timeouts and error normalization;
- idempotency support;
- current-state lookup;
- reconciliation;
- contract tests and version changes.

Do not expose vendor error strings directly as workflow semantics. Normalize states such as validation failure, authorization failure, conflict, throttled, transient, permanent, and outcome unknown.

Bulk backfills must not starve interactive reads or reconciliation.

## 12. Consistency and transactions

Use strong consistency where wrong order or duplication causes harm:

- one terminal approval transition;
- uniqueness of operation/idempotency key;
- expected resource version before execution;
- membership/authorization checks;
- active source version publication.

Use eventual consistency for rebuildable views such as search and analytics, but expose freshness and version.

Use a transactional outbox when domain state and an event must commit together. Consumers remain idempotent.

Avoid distributed two-phase commit across vendor systems. Use durable sagas/workflows with explicit compensation or reconciliation. Compensation is a business action, not automatic time travel.

## 13. Caching

Cache:

- public/static content;
- tenant configuration snapshots;
- bounded authorization-supporting data with explicit invalidation;
- safe retrieval results keyed by tenant, identity/scope, query, source/index versions, and policy version;
- model responses only when semantic equivalence, privacy, and freshness are defensible.

Never omit tenant, authorization, or active source version from a cache key.

Use TTL as a backstop. Publish invalidation on resource, membership, policy, deletion, and index changes. Prevent stale refill with version checks.

If a cache outage threatens the database, shed load and preserve high-value operations rather than stampeding the origin.

## 14. Reliability and overload

Set journey SLOs, not one global uptime number:

- authorized cited query success and latency;
- ingestion freshness;
- review queue deadline;
- action execution/reconciliation;
- deletion completion.

Patterns:

- deadlines and timeouts;
- bounded retries with jitter and budgets;
- circuit breakers;
- bulkheads by dependency and tenant;
- queue backpressure and admission control;
- per-tenant fairness;
- load shedding;
- last-known-good configuration within a lease;
- read-only or evidence-only degradation;
- kill switches for write capabilities.

Protect reconciliation and safety work during overload. A batch reindex or large tenant import should yield to interactive and uncertain-outcome processing.

## 15. Observability

Use one correlation chain across:

```text
request → retrieval → model → workflow → approval → connector → reconciliation
```

Record request/correlation ID, tenant-safe identifiers, resource/version, workflow/run, proposal/plan hash, operation ID, policy/config/model/index versions, and external reference.

Measure:

- user outcome and SLO;
- retrieval coverage and citation validity;
- model latency, tokens, cost, and fallback;
- workflow state age;
- queue lag and reviewer capacity;
- connector throttling and errors;
- duplicate suppression;
- outcome-unknown age;
- tenant fairness;
- deletion and retention;
- evaluation drift.

Do not put documents, prompts, PHI, credentials, or raw external payloads into general logs. Use restricted references and redaction by construction.

## 16. Evaluation and release

Evaluation layers:

1. deterministic unit/contract/security tests;
2. component evaluations for extraction and retrieval;
3. workflow evaluations for proposals, tools, approvals, and failures;
4. human evaluation with adjudication;
5. shadow production;
6. canary release;
7. continuous outcome monitoring.

Slice by tenant type, document quality, language, vendor, amount/risk, workflow, missing evidence, and attack pattern.

Release gates combine quality, safety, latency, cost, reliability, and sample confidence. Aggregate improvement cannot hide regression in a critical slice.

Version the release bundle:

```text
application + workflow + prompts + model + tool schemas
+ policies + configuration schema + parser/chunker
+ embedding/index + evaluation bundle
```

## 17. Security, privacy, and audit

Threat-model:

- cross-tenant access;
- prompt injection from documents;
- confused-deputy tool calls;
- credential leakage;
- malicious files;
- insecure webhooks;
- privilege revocation during workflows;
- support impersonation;
- sensitive logs and evaluation data;
- deletion resurrection.

Controls:

- server-derived identity and context;
- policy checks at read, proposal, approval, and execution;
- content treated as data, never authority;
- least-privilege tools;
- isolated parsing;
- encryption and workload identity;
- purpose limitation and minimum data;
- immutable minimal audit;
- retention/deletion manifests and tombstones;
- secret rotation;
- periodic access review.

An audit trail proves what the system did, with which versions and authority; it should not duplicate all sensitive evidence.

## 18. Disaster recovery

Classify state:

- **authoritative:** metadata, workflows, approvals, operation/outcome records, configuration/policy history;
- **immutable source:** object versions and restricted raw integration records;
- **rebuildable:** lexical/vector indexes, caches, analytics projections;
- **external:** ERP/project/clinical system state requiring reconciliation.

Define RPO/RTO per class. Back up authoritative state and keys with restore tests. Fence old writers during regional failover. Restore authoritative state first, then workflow state, then rebuild indexes.

After recovery:

- replay events with side effects disabled;
- preserve operation IDs;
- reconcile external outcomes;
- apply deletion tombstones;
- verify tenant and authorization boundaries;
- resume writes only after recovery criteria pass.

## 19. Cost model

Track cost per successful business outcome, not only per API call.

Components:

- object and database storage;
- OCR/parser compute;
- embeddings and indexing;
- model tokens or inference compute;
- workflow/broker/search infrastructure;
- observability and retention;
- third-party connector cost;
- reviewer time;
- support and incident load.

Optimize in order:

1. remove unnecessary work;
2. route deterministic queries away from models;
3. reduce context safely;
4. cache with correct scope/version;
5. batch low-urgency work;
6. use smaller approved models;
7. improve adoption so fixed cost produces outcomes.

A cheaper wrong action is not an optimization.

## 20. Construction end-to-end example

A project manager asks:

> Is invoice 8831 safe to approve, and can you code it to the ERP?

Flow:

1. Identity service resolves tenant, project membership, and role.
2. Query service fetches structured ERP invoice/PO data and retrieves authorized invoice/change-order evidence.
3. Answer cites immutable document pages and current source versions.
4. Deterministic checks compute arithmetic, duplicate candidates, PO balance, and required documents.
5. Model proposes cost code and explains uncertain evidence.
6. Policy routes the $135,000 invoice to project-manager and controller review.
7. Action plan binds amount, currency, vendor, project, invoice version, PO/change-order versions, code mapping, evidence, policy, tool schema, and operation ID.
8. A change order arrives; its new version invalidates the approval.
9. After reapproval, connector submits using stable idempotency identity.
10. ERP times out. Workflow shows “confirmation pending,” blocks blind retry, and reconciles by operation/reference ID.
11. Nightly batch compares platform operations with ERP ledger outcomes.
12. Metrics capture cycle time, correction, queue age, outcome-unknown duration, and duplicate avoidance.

## Healthcare operations variation

A discharge coordinator asks:

> Which patients need follow-up scheduling today?

The system retrieves authorized, current encounter and scheduling evidence. Operational rules identify missing appointments and deadlines. The model may summarize cited non-clinical context but cannot create clinical facts.

Role policy distinguishes schedulers, care coordinators, and clinicians. Scheduling actions may require patient/encounter version checks and approval based on local policy. Medication or clinical recommendations remain under licensed authority.

Late corrected records update the queue. PHI is minimized in prompts, logs, and evaluation data. If the clinical source or policy service is unavailable, the system holds consequential actions and presents a manual workflow with truthful freshness—not guessed advice.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Modular services | Independent scaling/ownership | Distributed complexity |
| Modular monolith first | Simpler deployment | Coarser scaling/failure boundary |
| Hybrid retrieval | Better exact + semantic recall | More tuning and indexes |
| Managed model | Fast capability | Provider/cost/privacy dependency |
| Self-hosted model | Control and predictable boundary | GPU and operations burden |
| Human approval | Risk control and feedback | Latency and staffing |
| Automatic action | Fast outcome | Higher error consequence |
| Strong consistency | Protects critical transitions | Latency/availability tradeoff |
| Eventual projections | Scalable and rebuildable | Freshness/reconciliation |
| Per-tenant isolation resources | Strong blast-radius control | Cost and operations |
| Shared multi-tenant resources | Efficiency | Noisy-neighbor/isolation complexity |
| Long event retention | Replayability | Cost and privacy exposure |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Model becomes source of truth | Fabricated state/action | Authoritative services and validation |
| Tenant filter after retrieval | Cross-tenant leakage | Pre-filter plus defense in depth |
| Partial index published | Incomplete answers | Atomic version publication |
| Cache omits policy/version | Stale unauthorized data | Complete scoped key/invalidation |
| Prompt injection controls tools | Confused-deputy action | Content-is-data and tool policy |
| Approval not plan-bound | Different action executes | Canonical plan hash |
| Role revoked after approval | Unauthorized write | Execution-time reauthorization |
| External timeout blindly retried | Duplicate business action | Outcome-unknown reconciliation |
| Replay triggers tools | Duplicate payment/message | Side-effect-free rebuild mode |
| One tenant consumes capacity | Others miss SLOs | Quotas and fair scheduling |
| Aggregate eval hides slice harm | Unsafe release | Critical-slice gates |
| Logs copy sensitive context | Privacy breach | Structured allowlist telemetry |
| Deletion omits vectors/caches | Residual exposure | Lineage-driven deletion |
| Failover permits two writers | Divergent workflows | Fencing/lease |
| Rollback ignores external effects | System and ERP disagree | Reconciliation |
| FDE remains sole operator | Unsustainable deployment | Runbooks, ownership, paired handoff |

## What to measure

Track user outcome, adoption funnel, query success/latency, freshness, authorized retrieval coverage, citation validity, insufficient-evidence rate, extraction quality, proposal correction, approval time, stale approvals, connector success, outcome-unknown age, reconciliation mismatch, duplicate suppression, queue lag, tenant fairness, model tokens/cost, cache behavior, evaluation slices, privacy/deletion SLA, incidents, recovery tests, onboarding time, and cost per completed outcome.

Every dashboard should let an operator move from a business symptom to the responsible workflow, source version, dependency, and owner.

## Interview prompt

> Design a multi-tenant AI operations assistant for construction and healthcare. It ingests documents and system records, answers cited questions, proposes actions, requires human approval for consequential work, and integrates with external systems that may throttle or time out after accepting a write. The design must support tenant isolation, long-running workflows, evaluation, observability, deletion, disaster recovery, and phased FDE deployment.

In 45 minutes, cover requirements, estimates, APIs/data model, architecture, ingestion, retrieval, agent/workflow boundaries, approvals, external writes, consistency, scaling, security, reliability, evaluation, operations, and tradeoffs.

## Worked answer

**One-line conclusion:** Build two controlled planes—an authorized, versioned evidence plane and a durable action plane—so AI can propose from cited truth while deterministic policy, human approval, idempotency, and reconciliation govern every consequential effect.

“I would start by clarifying read and action journeys, freshness, peak workload, prohibited outcomes, and SLOs. The platform stores authoritative tenant, resource/version, lineage, workflow, approval, operation, configuration, and audit state in a relational database; immutable files live in object storage; lexical/vector indexes are rebuildable projections.

Ingestion uses direct quarantine uploads plus signed webhooks, APIs, or CDC. It validates, parses, chunks, embeds, and atomically publishes a complete version. Queries authenticate, resolve server-side scope, pre-filter authorized hybrid retrieval, construct cited context, and call an approved model through a budgeted gateway. Exact balances and statuses come from structured authoritative sources.

I use durable workflows for state and agents only for bounded typed proposals. Consequential actions create a canonical plan bound to resource/evidence/policy/tool versions and reviewers. Execution rechecks current authorization and versions, then calls a connector with a stable operation ID. Timeouts become outcome unknown and reconcile before retry.

At-least-once delivery is made effectively once through event IDs, version guards, unique operation keys, and idempotent sinks. Per-tenant quotas and bulkheads isolate bursts. Degraded mode preserves authorized read-only evidence while write capabilities can be killed independently. Observability correlates query, retrieval, model, workflow, approval, connector, and reconciliation without logging sensitive content.

Releases use versioned bundles, offline and workflow evals, shadowing, canaries, critical-slice gates, and rollback plus external reconciliation. Backups restore authoritative state first; indexes rebuild; writers are fenced; deletion tombstones replay. I would deploy through an FDE pilot: offline evidence, live shadow, assisted cohort, then bounded automation after business, quality, security, and reliability gates pass.”

## Practice lab

Design and defend this system:

- 80 tenants, with one tenant generating 30% of document traffic;
- PDFs, email, ERP APIs, project webhooks, and healthcare operational events;
- cited questions need current authorization and p95 latency target;
- month-end ingestion is 5× normal;
- reviewers have role and separation-of-duties requirements;
- one action is approved, then its source record and approver role change;
- the ERP times out after a possibly accepted write;
- a model-provider outage begins during a queue spike;
- a tenant requests deletion while immutable backups remain;
- the primary region fails with workflows in flight;
- a new embedding model must launch without mixing vector spaces.

Give:

1. requirements and assumptions;
2. rough capacity equations;
3. APIs and data model;
4. high-level architecture;
5. read and write sequences;
6. tenant/security boundaries;
7. consistency and idempotency choices;
8. overload and degraded modes;
9. evaluation/release plan;
10. deletion and recovery;
11. three major tradeoffs;
12. a crisp one-line conclusion.

## References

- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
