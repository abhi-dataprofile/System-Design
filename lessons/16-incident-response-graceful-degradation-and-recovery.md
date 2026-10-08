# 16 — Incident response, graceful degradation, and recovery

Previous: [Deployment strategies, feature flags, and safe rollouts](15-deployment-strategies-feature-flags-and-safe-rollouts.md). Study time: 50–60 minutes including the exercise.

## Mental model

An incident is a period when actual or imminent user harm requires coordinated response. The objective is not to prove root cause immediately. It is to establish impact, stop the blast radius, preserve truthful state, restore the safest valuable capabilities, reconcile uncertainty, and learn.

Use two loops:

1. **Control:** detect → assess → contain → mitigate → recover → validate.
2. **Learning:** preserve evidence → explain contributing conditions → improve controls → verify them.

> Prefer a smaller truthful service over a fully available system that returns stale, unauthorized, unsupported, or duplicate outcomes.

Graceful degradation depends on business semantics. Cached contract data may be acceptable for browsing with a freshness warning; it is unsafe for approving payment when authorization or policy cannot be checked.

## When to fail open, fail closed, or degrade

| Capability | Failure | Safe posture | Reason |
| --- | --- | --- | --- |
| Public help | Search unavailable | Bounded cache | Low sensitivity/consequence |
| Authorized search | Authorization unavailable | Fail closed | Access cannot be established |
| Cited answer | Model unavailable | Evidence-only or defer | Do not invent synthesis |
| Invoice review | Retrieval degraded | Queue/manual review | Missing evidence changes decisions |
| ERP submission | Timeout after send | Outcome unknown; reconcile | Retry may duplicate |
| Approval write | Primary DB unavailable | Read-only; reject write | Transition cannot be durable |
| Clinical summary | Source stale | Label gap; clinician review | Silent incompleteness is unsafe |
| Medication/order action | Policy unavailable | Fail closed | Consequential action needs controls |

Choose these modes before an incident. Define staleness limits, allowed operations, entry/exit rules, user messages, and override authority.

## Severity and impact

Severity follows harm, not a dramatic graph:

- **SEV-1:** cross-tenant disclosure, unsafe clinical action, or widespread duplicate-payment risk;
- **SEV-2:** major workflow outage or reconciliation backlog threatening deadlines;
- **SEV-3:** limited degradation with a workaround;
- **SEV-4:** minor defect without material user impact.

Assess affected users, tenants, regions, workflows, risk classes, start time, growth, confidentiality/integrity/availability, safety, data corruption, deadlines, and external actions that are completed, pending, duplicated, or uncertain. Use the highest credible severity until evidence supports lowering it.

## Incident command

Assign explicit roles:

- **incident commander:** priorities, severity, decisions, handoffs;
- **operations lead:** containment, mitigation, recovery;
- **investigation lead:** hypotheses and evidence;
- **communications lead:** internal/customer updates;
- **scribe:** timeline, commands, observations, and decisions;
- **domain/security/privacy lead:** joins when impact requires expertise.

One person can hold several roles in a small team, but command remains clear. Use one channel, one current-status document, and one decision log. Separate observations from hypotheses.

At declaration record the incident ID, time, severity rationale, roles, affected capability/population, user behavior, suspected start, immediate containment, next update, and evidence links.

## Containment before root cause

Containment reduces harm while preserving evidence:

- disable a feature or release;
- activate a tool-specific kill switch;
- route new work to the last known good bundle;
- pause a consumer or request class;
- enter read-only mode;
- revoke credentials or isolate a tenant/region;
- cap concurrency and shed low-priority work;
- quarantine suspect inputs;
- pause payment, ERP, email, or clinical-action tools;
- snapshot traces, workflow state, versions, and audit evidence.

Avoid broad destructive actions. Restarting everything can erase evidence, cause a retry storm, and replay side effects. Rolling back traffic never reverses external writes.

## Graceful degradation modes

Design explicit product states:

```text
NORMAL
READ_ONLY
EVIDENCE_ONLY
QUEUE_FOR_LATER
MANUAL_REVIEW_REQUIRED
OUTCOME_UNKNOWN
TENANT_OR_REGION_ISOLATED
EMERGENCY_SHUTDOWN
```

Each state defines allowed reads/writes, freshness limits, status copy, telemetry, entry/exit authority, and treatment of queued and in-flight work.

“We saved your document and processing is delayed” differs from “We cannot confirm whether the ERP accepted this invoice.” Never collapse uncertainty into failure merely to simplify the UI.

## Load shedding and isolation

Prioritize:

1. safety, authorization, and reconciliation;
2. critical status and approved actions with deadlines;
3. interactive reads;
4. ingestion and indexing;
5. analytics, experiments, and bulk exports.

Reject overload early with retry guidance instead of accepting work that will time out. Use per-tenant quotas, bounded queues, deadlines, cancellation, and retry budgets. Backoff with jitter; unlimited retries amplify outages.

Circuit breakers stop repeated calls to an unhealthy dependency, moving through closed, open, and limited half-open probes. Bulkheads isolate queues, worker pools, connections, and budgets so reindexing cannot starve ERP reconciliation. These mechanisms protect capacity but do not decide whether incomplete output is semantically safe.

## AI-specific degradation

| Failure | Degraded behavior | Required guard |
| --- | --- | --- |
| Primary model unavailable | Evaluated fallback bundle | Compatible prompts/tools and quality gates |
| All models unavailable | Evidence-only or queued response | No synthetic answer |
| Vector search unavailable | Lexical search | Label limitations; evaluate affected queries |
| Reranker unavailable | Initial authorized ranking | Lower confidence and result cap |
| Citation validation unavailable | Withhold generated answer | Do not claim grounding |
| Tool executor unhealthy | Read-only copilot | Block action execution |
| Evaluation service unavailable | Keep proven bundle | Stop promotion, not serving |
| Injection detector unavailable | Deterministic policy/tools remain | Classifier is not sole boundary |

A smaller model is not automatically a safe fallback. Pre-evaluate model, prompt, schema, budgets, and allowed capabilities as a bundle. Never reuse a cached generated answer when authorization, evidence, prompt/model, or policy assumptions differ.

## Uncertain side effects

An external timeout after transmission creates:

```text
local fact: request sent
external fact: may have committed
safe state: OUTCOME_UNKNOWN
```

Response:

1. stop blind retries;
2. preserve the stable operation/idempotency key;
3. store request hash, target, attempt, and time;
4. query the external system by operation or business key;
5. resolve to succeeded, failed-before-commit, or manual review;
6. communicate uncertainty and final outcome;
7. inspect possible duplicates before replay.

If the dependency cannot deduplicate or query by key, use a reconciliation ledger and manual approval. At-least-once delivery is not permission to execute a financial or clinical action more than once.

## Recovery order

Restore:

1. identity, authorization, secrets, and time;
2. authoritative database and durable workflow state;
3. reconciliation and integrity checks;
4. critical APIs and status;
5. consumers at a controlled rate;
6. retrieval and model features;
7. bulk jobs and experiments.

Recovery can cause a second incident: cold caches, queued retries, reconnecting clients, and delayed events create surge. Ramp traffic gradually and protect tenant fairness. Do not drain a backlog at maximum speed without downstream budgets; track arrival rate, service rate, oldest age, and projected drain time.

## Validate recovery

Green infrastructure is insufficient. Verify:

- SLO burn stabilizes and end-to-end requests succeed;
- oldest workflow/queue age falls;
- database, replica, cache, and index invariants hold;
- authorization and tenant isolation pass;
- uncertain and duplicate operations are reconciled;
- degraded flags are intentionally reset;
- the active AI bundle passes quality checks;
- support reports agree with telemetry;
- the telemetry pipeline itself is current.

Observe through a defined stability window. Residual manual work needs a named owner and deadline.

## Communication

Updates state user impact, affected scope and start time, what is safe or unsafe to do, mitigation, known uncertainty, and the next update time. Avoid speculative root causes and premature “resolved.”

Example:

> Invoice review remains available, but ERP submission is paused. Requests sent between 10:14 and 10:31 ET may have been accepted without confirmation; do not resubmit them. We are reconciling each operation and will update by 11:15 ET.

## Construction example

A connector release causes timeouts after requests are transmitted. Messages redeliver and reconciliation falls behind.

1. action-integrity and oldest-reconciliation alerts declare SEV-2;
2. the commander assigns operations, investigation, communications, and scribe;
3. an ERP-write kill switch stops new submissions while invoice review stays available;
4. unsent work becomes `MANUAL_REVIEW_REQUIRED`; sent-but-unconfirmed work remains `OUTCOME_UNKNOWN`;
5. new traffic returns to the proven connector, but no operation is replayed;
6. an isolated reconciliation pool queries ERP by stable operation ID;
7. per-tenant quotas protect recovery capacity;
8. the UI tells controllers not to resubmit;
9. ERP identifiers, local state, duplicate checks, and falling queue age validate recovery;
10. follow-ups add a connector canary, game day, capacity target, and clearer status copy.

The incident remains open until uncertain operations are reconciled or transferred to owned manual review.

## Healthcare variation

A delayed clinical feed makes medication evidence stale. The assistant stops synthesized medication guidance, displays source timestamps, continues unaffected authorized access, and requires clinician review.

A patient-safety lead joins command. The team identifies affected encounters, preserves evidence versions, and checks whether drafts influenced care. Feed restoration alone is insufficient: current-version checks, patient reconciliation, clinical communication, and privacy/regulatory steps must complete.

Failing open preserves availability while increasing harm; evidence-only review is the safer degraded service.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Early declaration | Faster coordination | May overstate severity |
| Wait for certainty | Fewer false alarms | Larger blast radius |
| Broad shutdown | Stops many harms | Disrupts safe capabilities |
| Capability kill switch | Precise containment | Requires prior testing |
| Fail closed | Protects integrity | Availability loss |
| Fail open | Preserves availability | Stale/unauthorized behavior |
| Queue for later | Preserves intent | Backlog/deadline risk |
| Evidence-only mode | Truthful and safer | Less convenient |
| Automatic fallback | Fast continuity | Hidden quality regression |
| Automatic rollback | Rapid mitigation | Cannot undo side effects |
| Manual reconciliation | Resolves ambiguity | Slow and expensive |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Root-cause debate delays containment | Harm grows | Contain on credible impact |
| Everyone debugs independently | Conflicting actions | Command roles |
| Restart everything | Evidence loss/retry storm | Targeted reversible action |
| Fail open on authorization | Data exposure | Fail closed |
| Generic cached AI answer | Stale/unauthorized advice | Evidence-aware fallback |
| Untested fallback model | New safety/tool failures | Evaluated bundle |
| ERP timeout marked failed | Duplicate resubmission | Outcome-unknown state |
| Rollback assumed to undo writes | State divergence | Reconciliation ledger |
| Queue drains at full speed | Dependency re-collapses | Controlled priority ramp |
| Status says operational | Users remain blocked | Outcome-based status |
| Resolve at green graph | Corruption/backlog remains | End-to-end validation |
| Sensitive payload in logs | Secondary incident | Restricted minimal evidence |
| Residual work lacks owner | Cases abandoned | Ownership and deadline |

## Post-incident review

Document impact, timeline, contributing technical and organizational conditions, defenses that worked or failed, why the blast radius spread, reconciliation results, and prioritized actions with owners and verification.

Avoid “human error” as a root cause. Ask why the action was possible, reasonable, insufficiently reviewed, or hard to detect. Durable actions change systems: automated invariants, safer defaults, tested kill switches, capacity isolation, contracts, evaluations, or game days—not “be more careful.”

## What to measure

Track time to detect, declare, contain, mitigate, recover, and reconcile; affected users/workflows and severity-weighted harm; SLO burn; degraded-mode duration; oldest queue/workflow age and drain rate; queued, rejected, duplicate-suppressed, and outcome-unknown actions; security/data/quality violations; communication timing; rollback and kill-switch success; residual-case closure; and verified action-item completion.

## Interview prompt

> Design incident response and graceful degradation for a multi-tenant AI assistant that ingests construction documents, answers cited questions, reviews invoices, obtains approval, and submits to an ERP. A connector begins timing out after requests are sent, the queue retries, reconciliation falls behind, and one model provider is also degraded. Adapt the response for a healthcare discharge assistant with a delayed medication feed.

Cover severity, roles, containment, fail-open/fail-closed decisions, isolation, AI fallbacks, uncertain side effects, load shedding, recovery order, validation, communication, and prevention.

## Worked answer

**One-line conclusion:** Contain the harmful capability first, preserve truthful workflow states, restore safe functions by priority, and do not declare recovery until uncertain external effects and data integrity are reconciled.

“I would declare based on user and action-integrity impact, assign command, operations, investigation, communications, and scribe roles, and maintain one timeline. I would disable ERP writes with a tool-specific kill switch while keeping authorized invoice review available. New submissions queue visibly; already-sent calls become outcome unknown and are never blindly retried. Each keeps its stable operation ID, while an isolated reconciliation pool queries ERP. I would return new traffic to the proven connector, rate-limit tenants, shed bulk work, and reserve capacity for reconciliation and status. Model degradation uses only a pre-evaluated read-only fallback; without citation validation, the UI becomes evidence-only. Authorization failure fails closed. Recovery restores identity and durable truth first, then reconciliation, critical APIs, controlled consumers, and optional AI. I validate actions, invariants, queue age, quality, and user reports through a stability window. Communications state uncertainty, what users must not repeat, and the next update. The review produces tested canaries, kill switches, bulkheads, evaluation cases, and game days.”

## Practice lab

Respond to this incident:

- a connector release changes retry behavior;
- ERP timeouts begin after some requests transmit;
- 180 duplicate attempts occur, but local deduplication blocks 172;
- eight operations have no confirmed outcome;
- reconciliation age reaches 35 minutes;
- the primary model provider returns 429s;
- authorization remains healthy;
- two controllers manually resubmit invoices.

Define severity, roles, first three containment actions, degraded modes, request priorities, stable identifiers, duplicate/outcome reconciliation, model fallback, user message, recovery sequence, validation window, and five concrete prevention actions.

## References

- [Google SRE Book — Managing Incidents](https://sre.google/sre-book/managing-incidents/)
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)
- [Google SRE Book — Handling Overload](https://sre.google/sre-book/handling-overload/)
- [AWS Builders' Library — Avoiding fallback in distributed systems](https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/)
- [NIST Computer Security Incident Handling Guide](https://csrc.nist.gov/pubs/sp/800/61/r2/final)
