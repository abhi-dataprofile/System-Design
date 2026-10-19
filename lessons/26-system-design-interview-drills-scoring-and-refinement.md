# 26 — System-design interview drills, scoring rubric, and answer refinement

Previous: [End-to-end capstone: multi-tenant AI operations assistant](25-end-to-end-capstone-multi-tenant-ai-operations-assistant.md). Study time: 55–70 minutes including the drill.

## Mental model

A system-design interview is a collaborative decision-making exercise under uncertainty. The interviewer is evaluating whether you can turn an ambiguous outcome into a coherent system, expose the hard parts, and defend tradeoffs.

Use this loop:

```text
frame → quantify → decompose → trace → stress → decide → summarize
```

A strong answer is not the architecture with the most components. It is the smallest design that satisfies the stated requirements, has explicit sources of truth, and remains understandable during failure.

> Make assumptions visible, connect each component to a requirement, and spend depth on the highest-risk boundary rather than drawing a generic cloud diagram.

Your goal is to leave the interviewer with a clear one-line conclusion: the core design decision and why it fits.

## When to use this framework

Use it for:

- 30–60 minute system-design interviews;
- Forward Deployed Engineer customer scenarios;
- AI/ML platform and agent-system design;
- API and integration architecture discussions;
- production-readiness and debugging follow-ups;
- take-home designs that need concise rationale;
- self-review of an existing architecture.

Adapt the time allocation to the interview. Do not rigidly recite a script when the interviewer signals a desired deep dive.

## What interviewers are testing

Most prompts test a combination of:

1. requirement discovery;
2. prioritization and scope control;
3. quantitative reasoning;
4. data and API modeling;
5. decomposition and ownership;
6. distributed-systems correctness;
7. security and tenant isolation;
8. AI-specific grounding and evaluation;
9. reliability and operations;
10. communication and tradeoff judgment.

An FDE prompt adds customer ambiguity, existing-system constraints, adoption, phased delivery, and operational handoff.

A correct answer can use different technologies. Weak reasoning cannot be fixed by naming fashionable products.

## The first two minutes

Open with:

1. restate the outcome;
2. name primary actors;
3. ask the highest-leverage questions;
4. propose a scope;
5. state the leading risk.

Example:

> “I’ll design a multi-tenant assistant that answers cited invoice questions and proposes ERP coding, with humans authorizing consequential writes. I’ll first clarify freshness, scale, source-of-truth systems, approval policy, and what must never happen. I’ll keep autonomous payment out of scope unless we establish stronger requirements.”

Good clarifying questions:

- What are the top two user journeys?
- Which system is authoritative for each entity?
- What freshness and latency matter?
- What is the expected and peak scale?
- Which actions are reversible or consequential?
- Who may view, approve, and execute?
- What is the acceptable behavior during dependency failure?
- Which existing systems and APIs are constraints?
- What must never happen?
- Is this a greenfield product, a pilot, or a migration?

Do not spend ten minutes interviewing the interviewer. Make reasonable assumptions, state them, and continue.

## Time-box a 45-minute answer

| Minutes | Focus | Output |
| --- | --- | --- |
| 0–5 | Requirements and scope | actors, journeys, SLOs, non-goals |
| 5–8 | Estimates | traffic, storage, bottlenecks |
| 8–13 | APIs and data | contracts, entities, authority |
| 13–23 | High-level design | major boundaries and flows |
| 23–33 | Deep dive | highest-risk path |
| 33–39 | Failure, scale, security | stress cases and mitigations |
| 39–43 | Tradeoffs and evolution | alternatives, rollout |
| 43–45 | Summary | decisions, risks, conclusion |

If the interviewer interrupts, that is collaboration, not failure. Answer the question, update assumptions, and return to the backbone.

## Scope with a requirement table

Keep a compact table:

| Category | Must have | Later/non-goal |
| --- | --- | --- |
| Users | project managers, controllers | public access |
| Read | cited invoice/PO/change-order evidence | general web search |
| Write | approval-bound ERP coding | autonomous payment |
| Quality | source/version citations | unsupported inference |
| Security | tenant/project authorization | client-provided tenant trust |
| Reliability | durable workflows, reconciliation | synchronous cross-system transaction |
| AI | bounded proposal and summarization | model as source of truth |

This prevents the architecture from drifting toward unrequested capabilities.

## Estimate only what changes the design

Useful calculations answer a design question:

- peak query rate → service and model concurrency;
- daily pages → parser/embedding throughput;
- active corpus → index size and sharding;
- token volume → provider quota and cost;
- proposed actions → review capacity;
- connector rate limit → queue and drain time;
- event retention → replay feasibility;
- tenant skew → fairness and isolation.

State variables when numbers are missing:

[
	ext{peak QPS} =
rac{	ext{daily users} 	imes 	ext{queries/user/day}}
{	ext{active seconds/day}}
	imes 	ext{burst factor}
]

[
	ext{reviewer hours/day} =
	ext{review cases/day} 	imes 	ext{minutes/case} / 60
]

Do not calculate arbitrary storage to perform sophistication. Say which threshold would change your choice.

## Draw boundaries before products

Name responsibility first:

- edge and identity;
- application/query API;
- metadata and workflow state;
- object/source storage;
- ingestion pipeline;
- retrieval/index layer;
- model gateway;
- policy and configuration;
- review queue;
- connectors;
- observability/evaluation.

Then choose implementation options if useful.

Prefer:

> “A durable broker decouples ingestion and supports replay; Kafka, Pub/Sub, or another managed log could fit depending on existing platform.”

Avoid:

> “I use Kafka because it scales.”

The boundary and required semantics matter more than the logo.

## Trace two end-to-end flows

Always trace at least one read and one write.

### Read flow

```text
authenticate → authorize scope → retrieve current sources
→ rank → construct cited context → model → validate → respond
```

Name:

- source of truth;
- cache key and invalidation;
- authorization point;
- freshness/version;
- insufficient-evidence behavior;
- latency budget;
- observable IDs.

### Write flow

```text
proposal → policy → review → immutable approval
→ execution-time revalidation → external write
→ confirmation or reconciliation
```

Name:

- state machine;
- transaction boundary;
- resource version;
- idempotency key;
- outcome-unknown behavior;
- user-visible status;
- audit evidence.

A design is incomplete if it only traces successful reads.

## Choose the deep dive

Select the boundary with the greatest combination of uncertainty and consequence.

Common choices:

- snapshot plus CDC handoff;
- tenant-safe retrieval;
- approval binding and stale inputs;
- external timeout after possible acceptance;
- long-running workflow recovery;
- model evaluation and release;
- regional failover with in-flight writes;
- deletion propagation across AI artifacts.

Tell the interviewer why:

> “I’ll go deeper on the ERP write path because duplicate or stale financial actions are more consequential than a temporarily slow answer.”

Depth means states, IDs, versions, failure points, and recovery—not extra boxes.

## The source-of-truth test

For every important noun ask:

- Who owns it?
- What is its stable ID?
- Is it mutable or versioned?
- What freshness is required?
- What happens on conflict?
- Can the derived copy rebuild?
- How is deletion represented?

A common weak answer puts “data lake” or “vector DB” at the center without naming the authoritative invoice, approval, patient encounter, membership, or external outcome.

## The transaction-boundary test

Ask what must commit atomically.

Examples:

- workflow state plus outbox event;
- idempotency record plus local state transition;
- complete index manifest plus active-version pointer;
- reviewer decision plus terminal case transition.

Do not claim an atomic transaction across your database and an external ERP. Use a durable operation, stable identity, explicit uncertain state, and reconciliation.

## The failure matrix

Stress the design across five dimensions:

| Failure | Question |
| --- | --- |
| Omission | What if an event/file never arrives? |
| Duplication | What if delivery or submission repeats? |
| Reordering | What if an older version arrives later? |
| Delay | What if the dependency responds after timeout? |
| Corruption | What if data is malformed or semantically wrong? |

Then test infrastructure:

- database unavailable;
- queue backlog;
- cache failure;
- model/provider throttling;
- search index stale;
- policy/config unavailable;
- region failure;
- credentials revoked;
- tenant overload.

For each failure explain: detection, containment, user-visible state, retry/repair, reconciliation, and recovery evidence.

## The AI-specific test

A production AI design must answer:

- Which facts are authoritative?
- How is authorization applied before retrieval?
- What source/version lineage reaches the answer?
- What happens when evidence conflicts or is missing?
- Which output is structured and validated?
- Why is a model needed instead of deterministic logic?
- Are tools read-only or side-effecting?
- How are prompt injection and data leakage contained?
- What offline and online evaluations gate release?
- What are latency, token, and cost budgets?
- How are prompt/model/index versions rolled out and rolled back?
- Where does human judgment remain necessary?

“Use RAG to prevent hallucinations” is not sufficient. Retrieval can return stale, unauthorized, or irrelevant evidence.

## The multi-tenant test

State tenant isolation early, not as an afterthought.

Check:

- server-derived tenant context;
- database/object/index/queue keys;
- cache and idempotency keys;
- connector credentials;
- configuration and policy scope;
- logs, evaluations, support tools, and replay;
- quotas and noisy-neighbor controls;
- deletion and regional residency.

Explain the isolation model: shared schema with guarded tenant keys, schema/database per tenant, or hybrid tiers. Tie it to scale, cost, compliance, and operational burden.

## The human-workflow test

For review systems explain:

- why review is required;
- routing by consequence and uncertainty;
- evidence packet;
- qualified role and separation of duties;
- atomic claim or collaborative quorum;
- approval plan/version binding;
- stale decision handling;
- queue capacity and SLA;
- reviewer feedback quality;
- overload behavior.

“Human in the loop” is not a design until the queue, authority, evidence, and execution boundary are specified.

## Tradeoffs must be contextual

Use this structure:

> “I choose X because requirement Y dominates. It costs Z. If condition C changes, I would switch to alternative A.”

Example:

> “I choose a shared multi-tenant database with tenant IDs in every key because there are hundreds of small tenants and operational simplicity matters. It increases isolation risk, so I add server-derived scope, row policies, tenant-aware tests, and per-tenant quotas. For regulated high-volume tenants, I would offer isolated data-plane resources.”

Avoid empty comparisons such as “SQL is consistent, NoSQL is scalable.”

## Communicate uncertainty well

Strong language:

- “I’m assuming…”
- “The source system must confirm…”
- “This is a derived projection, not truth.”
- “The external outcome is unknown until reconciled.”
- “I would measure this before choosing…”
- “The design fails closed here because…”
- “This is outside the first release.”

Weak language:

- “Exactly once everywhere.”
- “The AI knows…”
- “We can just retry.”
- “Kafka solves it.”
- “Security can be added later.”
- “The system will never fail.”

Precision increases credibility.

## Scoring rubric: 100 points

### 1. Framing and requirements — 12 points

- 0–3: repeats prompt; no scope;
- 4–7: identifies actors and features;
- 8–10: prioritizes journeys, SLOs, risks, non-goals;
- 11–12: connects business outcome, prohibited outcomes, and evolution.

### 2. Estimates and bottlenecks — 8 points

- 0–2: no scale reasoning;
- 3–5: rough traffic/storage;
- 6–7: estimates workload-specific bottlenecks;
- 8: identifies the threshold that changes the architecture.

### 3. Data, APIs, and ownership — 12 points

- 0–3: generic stores/endpoints;
- 4–7: useful entities and APIs;
- 8–10: source of truth, versions, contracts, idempotency;
- 11–12: clear transaction and lineage boundaries.

### 4. Architecture and flows — 16 points

- 0–5: boxes without responsibilities;
- 6–10: coherent components and happy path;
- 11–14: complete read/write flows and state transitions;
- 15–16: minimal, evolvable boundaries tied to requirements.

### 5. Reliability and distributed correctness — 14 points

- 0–4: generic retries;
- 5–9: timeouts, queues, replication, recovery;
- 10–12: duplicates, order, backpressure, uncertain outcomes;
- 13–14: explicit invariants, reconciliation, and evidence of recovery.

### 6. Security, privacy, and tenancy — 12 points

- 0–3: encryption/RBAC buzzwords;
- 4–7: authn/authz and tenant filters;
- 8–10: trust boundaries, least privilege, lifecycle;
- 11–12: execution-time policy, support/replay/evaluation isolation.

### 7. AI and human workflow — 12 points

- 0–3: model call only;
- 4–7: retrieval and basic guardrails;
- 8–10: lineage, evaluation, typed tools, approval;
- 11–12: risk routing, release gates, stale-plan handling, outcomes.

### 8. Operations, cost, and delivery — 8 points

- 0–2: deployment only;
- 3–5: observability, rollout, cost;
- 6–7: SLOs, degradation, rollback, ownership;
- 8: phased evidence, handoff, cost per outcome.

### 9. Communication and tradeoffs — 6 points

- 0–1: rambling or overly terse;
- 2–3: understandable;
- 4–5: structured, responsive, contextual tradeoffs;
- 6: crisp synthesis with explicit uncertainty and conclusion.

Interpretation:

- **85–100:** senior-quality, coherent and defensible;
- **70–84:** strong, with a few incomplete boundaries;
- **55–69:** reasonable foundation but shallow failure/security reasoning;
- **below 55:** generic architecture or major correctness gaps.

Score evidence, not confidence or diagram beauty.

## Self-critique after a drill

Record your answer, then review in this order:

1. What was my one-line conclusion?
2. Did I establish scope and prohibited outcomes?
3. Could I name the source of truth for every key entity?
4. Did I trace read and write paths?
5. Did I identify the most consequential failure?
6. Did I use IDs, versions, states, and transaction boundaries?
7. Did I explain tenant/security enforcement at execution?
8. Did my AI component have a justified job and evaluation plan?
9. Did each tradeoff connect to a requirement?
10. Did I finish with operations, rollout, and handoff?

Mark unsupported claims in red, vague components in yellow, and clear decisions in green.

## Refinement method

Do not rewrite the entire answer after every drill. Improve the weakest boundary.

### Pass 1 — Structure

Reduce to:

- requirements;
- estimates;
- APIs/data;
- architecture;
- read flow;
- write flow;
- deep dive;
- failures/security;
- tradeoffs;
- conclusion.

### Pass 2 — Correctness

Add missing:

- authoritative owner;
- stable ID;
- version;
- state machine;
- idempotency;
- failure state;
- reconciliation.

### Pass 3 — Specificity

Replace:

> “Use a queue for scalability.”

With:

> “Durably enqueue ingestion after upload completion so parser retries do not hold the request; messages are at-least-once, keyed by tenant/resource version, and consumers publish only a complete version.”

### Pass 4 — Compression

Remove repeated principles. Keep one strong example for each.

### Pass 5 — Adversarial follow-up

Ask:

- What if the retry arrives after success?
- What if access is revoked mid-workflow?
- What if the index and database disagree?
- What if one tenant consumes all capacity?
- What if rollback cannot undo an external write?
- What if the model is unavailable?
- What if a deletion request overlaps backup recovery?

A good refined answer becomes shorter and more precise.

## Common weak answers and repairs

| Weak answer | Why it fails | Repair |
| --- | --- | --- |
| “Use microservices” | No boundary rationale | Start modular; split by ownership/scaling |
| “Use Kafka for scale” | Missing delivery semantics | Define event ID, partition, replay, sink |
| “Exactly once” | External effects ignored | Idempotency plus reconciliation |
| “RAG prevents hallucination” | Retrieval failure ignored | Authorization, versions, citations, eval |
| “Human approves” | No review system | Roles, evidence, queue, plan binding |
| “Encrypt everything” | No threat model/key design | Specify boundary, keys, access, rotation |
| “Cache responses” | Stale/cross-tenant risk | Scoped versioned keys and invalidation |
| “Autoscale” | Wrong bottleneck | Work units, quotas, headroom, admission |
| “Multi-region” | Consistency omitted | RPO/RTO, writer fencing, recovery order |
| “Roll back” | External effects persist | Rollback software; reconcile business state |
| “We’ll monitor it” | No actionable signals | SLO, alert, owner, runbook |
| “Build full automation” | Unvalidated risk/adoption | Offline → shadow → assisted → bounded automation |

## Construction example

Prompt:

> Design an invoice operations assistant for a contractor. It ingests invoices, purchase orders, and change orders; answers cited questions; proposes cost codes; and writes approved coding to an ERP.

Strong framing:

- ERP owns invoice/payment truth;
- project system owns current change orders;
- documents are immutable versioned evidence;
- workflow service owns proposals and approvals;
- assistant starts with cited decision support;
- autonomous payment is out of scope.

Highest-risk deep dive: ERP write after approval.

Explain:

1. normalized plan binds invoice amount/currency/vendor/project, resource versions, cost-code mapping, policy/tool versions, and expiry;
2. project manager and controller decisions are immutable;
3. worker rechecks roles and source versions;
4. stable operation ID guards retries;
5. timeout becomes `OUTCOME_UNKNOWN`;
6. reconciliation queries ERP by external reference;
7. user sees “confirmation pending,” not “failed”;
8. month-end burst uses fair queues and preserves reconciliation capacity.

A polished construction answer connects this technical path to cycle time, correction rate, approval SLA, and duplicate avoidance.

## Healthcare operations variation

Prompt:

> Adapt the design for discharge follow-up coordination.

State boundaries first:

- clinical record remains authoritative;
- the assistant manages operational evidence and queues;
- schedulers cannot approve clinical recommendations;
- patient/encounter authorization is server-derived;
- PHI is minimized in prompts, logs, and evaluation;
- late corrections revise the queue;
- clinical actions require current licensed authority.

Choose a deep dive such as current authorization plus stale clinical record. If the patient record changes after a recommendation, invalidate the proposal. If policy or clinical source is unavailable, hold consequential action and provide a truthful manual workflow.

Do not describe a scheduling optimization as if it grants medical authority.

## Interview prompt

> You have 45 minutes to design a multi-tenant AI assistant for construction invoice operations. It must ingest documents and ERP/project data, answer cited questions, propose cost coding, route approvals, and execute approved ERP writes. One tenant creates 35% of traffic, scanned documents vary in quality, roles can change mid-workflow, and the ERP may time out after accepting a write. The system must support healthcare operations later.

Answer using the scoring rubric. Prioritize two user journeys, estimate scale with variables, name sources of truth, trace read/write flows, deep-dive on the hardest boundary, inject three failures, and finish with tradeoffs and a one-line conclusion.

## Worked answer

**One-line conclusion:** Separate an authorized, versioned evidence plane from a durable action plane so AI can propose from cited truth while policy, human approval, idempotency, and reconciliation control every ERP effect.

“I will scope the first release to cited invoice questions and approval-bound cost coding; autonomous payment is a non-goal. The ERP owns invoice, PO, payment, and posting state; the project system owns current change orders; object storage holds immutable documents; the workflow database owns proposals, approvals, and operation outcomes.

I would estimate peak query concurrency, daily pages, embedding throughput, active index size, reviewer minutes, and ERP quota, including a month-end burst and the 35% tenant. APIs expose resource versions and asynchronous operation IDs. Uploads go to quarantine, then bounded parsing, extraction, deterministic chunking, embedding, and atomic index publication. Query requests resolve server-side tenant/project scope, pre-filter hybrid retrieval, combine structured ERP facts with cited document evidence, and use a model gateway with typed output and insufficient-evidence behavior.

Cost coding is a durable workflow. A normalized plan binds tenant, invoice/version, amount, currency, vendor, project, current PO/change-order versions, mapping, evidence, policy/tool versions, expiry, and operation ID. Qualified reviewers approve that hash. Execution rechecks role and versions. Changed access or evidence makes approval stale. The connector uses the stable ID; an ERP timeout becomes outcome unknown and reconciles before retry.

Queues use per-tenant limits and weighted fairness; reconciliation retains protected capacity. Model outage degrades to structured evidence and manual review, not blocked ERP truth. Observability correlates query, retrieval, model, workflow, approval, operation, and external reference. Evaluations slice by scan quality, vendor, amount, missing evidence, and tenant. I would deploy offline, shadow, assisted, then bounded automation. Healthcare reuses the evidence and workflow primitives but enforces patient scope, minimum PHI, and licensed clinical authority.”

## Practice drill

Record a 12-minute answer to the interview prompt without reading the worked answer.

Then:

1. score yourself out of 100;
2. list the three lowest rubric categories;
3. identify one unsupported claim;
4. identify one missing source of truth;
5. identify one missing failure state;
6. rewrite only the weakest two-minute section;
7. deliver a 30-second conclusion;
8. adapt the design to healthcare in 90 seconds.

Repeat the same prompt two days later. Improvement means clearer decisions and stronger failure reasoning—not more component names.

## References

- [Google Cloud Architecture Framework — System design](https://cloud.google.com/architecture/framework/system-design)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
