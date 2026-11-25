# Human-AI workflow orchestration, case management, and exception operations

A durable workflow coordinates known steps. A review queue allocates human attention. A saga coordinates distributed effects. A **case-management system** combines these mechanisms around a business outcome whose path may change as evidence, policy, and reality change.

Examples include resolving a disputed construction invoice, closing an RFI, handling a failed inspection, coordinating discharge, or correcting a denied claim. These are not single requests. They may run for days, require several roles, wait on external parties, fork into parallel work, and end differently depending on evidence.

## Mental model: the case is the durable unit of operational truth

Model the system around a case record, not around a model conversation or one background job:

```text
business trigger
  -> case + policy snapshot + evidence manifest
  -> milestones, tasks, timers, decisions, and child workflows
  -> normal path or explicit exception state
  -> authorized effects and outcome reconciliation
  -> closure evidence, metrics, and retained audit trail
```

A case answers:

- What outcome is being pursued?
- Who owns it now?
- Which state and deadline apply?
- Which evidence and policy versions support the current decision?
- Which tasks are ready, blocked, claimed, completed, or canceled?
- Which AI outputs are proposals rather than facts?
- Which external actions were intended, attempted, confirmed, or left outcome-unknown?
- Why was the case closed, and can it be reopened safely?

The workflow engine knows **how execution progresses**. The case record explains **what the business is trying to resolve**.

## When to use it

Use case management when work has several of these properties:

- it lasts longer than one request or process lifetime;
- people and systems collaborate across roles or organizations;
- documents and facts arrive incrementally;
- the next valid step depends on judgment or newly discovered evidence;
- deadlines, escalations, reminders, or legal holds matter;
- exceptions are common enough to be first-class states;
- several child workflows contribute to one outcome;
- managers need a complete operational view;
- work may pause, transfer, reopen, or be superseded.

Use a fixed workflow when the path is stable and exceptions are rare. Use an ordinary database transaction for a small atomic state change. Use a queue for asynchronous delivery, not as the authoritative case model.

Do not create a generic workflow platform for every CRUD operation. The operational and migration cost is justified only when durable coordination and visibility matter.

## Case, workflow, task, decision, and event

Keep the core concepts separate:

| Concept | Meaning | Example |
|---|---|---|
| Case | Business outcome and its complete operational context | Resolve disputed change-order invoice |
| Workflow instance | Executable path within the case | Validate invoice and route approval |
| Task | Bounded unit assigned to a human or service | Confirm installed quantity |
| Decision | Versioned judgment or policy result | Escalation required above threshold |
| Event | Immutable occurrence | Owner replied; ERP post confirmed |
| Timer | Durable future trigger | Escalate after two business days |
| Evidence | Versioned source supporting a claim | Contract, drawing, photo, email |
| Effect | Attempt to change an external system | Create ERP hold |
| Exception | State requiring alternate handling | Contract amount conflicts with ERP |

A case may contain multiple workflows. A workflow retry must not create a second case, duplicate a task, or repeat an external effect.

## Start with the business state machine

Do not derive state from scattered task rows. Define explicit case states and allowed transitions.

A disputed-invoice case could use:

```text
OPEN -> TRIAGE -> GATHERING_EVIDENCE -> DECISION_READY
     -> APPROVED | REJECTED | NEEDS_NEGOTIATION
     -> EXECUTING -> RECONCILING -> CLOSED
```

Orthogonal flags may represent legal hold, security incident, external dependency, or SLA breach, but avoid an uncontrolled combination of booleans. If combinations affect behavior, model them deliberately.

For each transition define:

- actor or service allowed to initiate it;
- required prior state;
- required evidence and freshness;
- policy version;
- commands and events;
- timers created or canceled;
- child workflows started or terminated;
- external effects permitted;
- compensating or reopening path;
- audit fields.

Use optimistic concurrency or compare-and-set on the case version so two actors cannot silently overwrite each other.

## Milestones, plans, and ad hoc work

A rigid step sequence is not enough for knowledge work. Separate:

- **milestones:** outcome-relevant checkpoints such as “evidence complete”;
- **required tasks:** must complete before a transition;
- **optional tasks:** created when context requires them;
- **discretionary tasks:** authorized operators may add them;
- **entry criteria:** facts enabling a stage;
- **exit criteria:** facts required to declare a milestone complete;
- **sentries:** events or facts that activate a task or stage.

This lets a project manager request a site verification without editing the core workflow definition. The task still needs a type, authority, due date, evidence contract, and completion schema.

Ad hoc must not mean ungoverned.

## Data model

A minimal case model includes:

- immutable case ID and tenant;
- case type and schema version;
- business key with uniqueness rules;
- current state, state version, and lifecycle timestamps;
- owner, team, watchers, and escalation path;
- priority and reason;
- policy and workflow-definition versions;
- evidence manifest with source version and authorization;
- facts separated from claims and model proposals;
- milestones and task references;
- deadlines, timers, pauses, and calendar;
- decisions, approvals, overrides, and reasons;
- child workflow and external-operation IDs;
- outcome, closure code, and reopen count;
- retention, legal hold, and deletion status.

Store large documents in an authorized evidence store, not in workflow history. Persist stable references, digests, and the minimum replay-safe attributes.

## Durable workflow execution

Workflow workers may crash, events may duplicate, and code may change while cases remain active.

The orchestrator should provide:

- durable timers and signals;
- at-least-once activity execution;
- deterministic replay or explicit persisted transitions;
- retry policies by error class;
- cancellation and timeout semantics;
- idempotent activities;
- versioning for in-flight instances;
- visibility into pending work;
- repair tools with audit trails.

A workflow history is not automatically the business audit record. It may be optimized for replay, compacted, or contain technical details. Project intentional case events into an auditable operational ledger.

Never rely on an in-memory sleep for a multi-day deadline.

## Work queues, ownership, and leases

Tasks should have:

- required skill, role, tenant, region, and separation-of-duties constraints;
- priority derived from policy, not arbitrary client input;
- created, available, due, and escalation times;
- owner or pool;
- lease with heartbeat and expiry;
- claim token and task version;
- evidence packet version;
- completion and rejection schemas.

A worker who loses a lease must not submit a late decision over newer work. Revalidate task version, case state, evidence, identity, and policy at completion.

Support pull, push, or hybrid assignment. Measure oldest age and deadline risk, not only queue length. Aging can prevent starvation, while protected capacity preserves urgent work.

## Business time and SLA clocks

“Two days” may mean 48 elapsed hours or two business days excluding weekends and holidays. Define:

- calendar and timezone;
- business hours;
- start event;
- pause conditions;
- response versus resolution SLA;
- warning and breach thresholds;
- responsibility while waiting on a customer;
- behavior when calendar versions change.

Persist the calculated deadline and the calendar version used. Do not recompute historical deadlines silently.

Timers should emit idempotent commands. A late timer after case closure must be harmless.

## Exception taxonomy

“Failed” is too vague. Classify exceptions:

| Type | Example | Default response |
|---|---|---|
| Validation | Missing contract number | Return for correction |
| Business conflict | Invoice exceeds approved amount | Evidence review |
| Policy exception | Requested override exceeds authority | Escalate |
| Dependency unavailable | ERP or document system down | Retry with deadline |
| Outcome unknown | ERP timed out after submission | Reconcile before retry |
| Capacity | No qualified reviewer before SLA | Rebalance or escalate |
| Security/privacy | Cross-tenant evidence suspected | Freeze and investigate |
| Model uncertainty | Evidence contradictory or out of distribution | Abstain and route |
| Data quality | OCR totals disagree with source | Re-extract or inspect |
| External waiting | Vendor has not responded | Pause/notify/escalate |

Give each exception an owner, severity, deadline, permitted remedies, and closure evidence. Dead-letter queues are technical holding areas; they are not an operator experience or a business resolution strategy.

## AI as a bounded case collaborator

AI can help:

- classify intake and propose a case type;
- extract facts with citations;
- identify missing evidence;
- draft summaries and communications;
- recommend next tasks from an allowed catalog;
- predict deadline risk;
- detect duplicate or related cases;
- suggest a decision with confidence and counterevidence.

The model must not directly mutate authoritative state. A deterministic layer validates:

- case and tenant scope;
- allowed proposal schema;
- evidence citations;
- current case version;
- role and policy;
- action risk;
- approval requirements;
- tool arguments.

Store prompt, model, retrieval snapshot, output, validation results, and reviewer disposition. Treat emails, PDFs, notes, and retrieved text as untrusted data that cannot redefine workflow policy.

A model conversation may expire; the case must remain complete without it.

## Evidence packets and decision records

A reviewer should receive a reproducible packet:

- decision question;
- current state and deadline;
- relevant facts and their sources;
- contradictory or missing evidence;
- policy and threshold;
- AI proposal labeled clearly;
- prior decisions and overrides;
- exact action that approval would authorize.

Bind the decision to a digest of the packet, case version, action payload, actor, and expiry. Before execution, verify that none of these changed. If material evidence arrives later, invalidate the decision and return the case to the appropriate state.

## Child workflows and process boundaries

A case might start child workflows for document collection, inspection, approval, ERP posting, notification, or reconciliation.

Define for each child:

- stable operation and correlation IDs;
- parent case and purpose;
- input snapshot;
- result schema;
- cancellation behavior;
- retry and timeout policy;
- effect ownership;
- whether the parent waits, proceeds, or compensates.

Avoid one giant workflow that couples every department and deployment. Also avoid uncoordinated choreography that makes it impossible to answer who owns the outcome. Keep local processes autonomous while projecting their relevant states into the case.

## Change management for long-running cases

Some cases outlive several software releases.

Version:

- case schema;
- state machine;
- workflow definition;
- policy bundle;
- task and decision schemas;
- model/prompt bundle;
- evidence contract.

Choose explicitly among:

- leave in-flight cases on the old version;
- migrate at a safe state;
- dual-read or compatibility layer;
- operator-assisted repair;
- terminate and recreate only when business semantics permit.

Test migrations using representative histories and edge states. Record old state, migration code version, new state, validation result, and approver. Never rewrite history to pretend the case always used the new policy.

## Reopening, merging, splitting, and superseding

Real work is not always one-to-one.

- **Reopen:** create a new lifecycle segment while preserving prior closure.
- **Merge:** choose a surviving case, preserve aliases, move or link tasks carefully, and prevent duplicate effects.
- **Split:** create child cases with explicit evidence and responsibility boundaries.
- **Supersede:** close an obsolete case with a pointer to its replacement.
- **Duplicate:** link cases and decide which operations remain valid.

These operations require authorization, concurrency control, audit events, and SLA rules. Never delete a duplicate case if it contains decisions or external effects.

## Observability and operational control

Track:

- cases opened, closed, reopened, merged, and aged;
- time in each state and waiting reason;
- response and resolution SLA attainment;
- task claim, lease expiry, reassignment, and rework;
- exception volume by type and source;
- automation, abstention, review, override, and acceptance rates;
- external-effect attempts, duplicates, and unknown outcomes;
- AI citation validity and proposal acceptance by slice;
- backlog by tenant, priority, skill, and owner;
- reopened cases and downstream corrections;
- deployment/version cohorts.

Use distributed traces for execution, but use case events and business metrics for operational truth. High task throughput can coexist with cases that never close.

## Failure modes

| Failure mode | Defense |
|---|---|
| Case state inferred from tasks | Explicit versioned state machine |
| Same intake creates duplicate cases | Business-key policy and idempotent creation |
| Two reviewers complete the same task | Lease, claim token, version check |
| Approval executes after evidence changes | Packet digest, expiry, execution revalidation |
| Multi-day timer disappears on restart | Durable timer in workflow engine |
| Late timer escalates a closed case | State/version check and idempotent command |
| “Waiting” hides ownership | Waiting reason, responsible party, next deadline |
| Technical retry appears as new business work | Stable task and operation IDs |
| DLQ becomes permanent backlog | Business exception queue with owner and remedy |
| AI changes policy through prompt injection | Trusted policy outside model context |
| AI summary omits contradictory evidence | Citation coverage and deterministic packet sections |
| One huge workflow blocks deployments | Bounded child workflows and versioned interfaces |
| New release breaks old cases | In-flight version strategy and migration tests |
| Manual repair edits data without trace | Audited repair command and reason |
| Merge repeats an external action | Preserve operation ledger and reconcile |
| SLA pauses indefinitely | Governed pause reasons and maximum review |
| Queue optimizes throughput, starves complex cases | Aging, protected capacity, cohort metrics |
| Closure records activity, not outcome | Required outcome and closure evidence |
| Reopened case erases prior accountability | New lifecycle segment with linked history |

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Fixed workflow | Predictable and testable | Brittle for knowledge work |
| Case model with discretionary tasks | Flexible and operator-friendly | More policy and UI complexity |
| Central orchestrator | Clear ownership and visibility | Coupling and control-plane dependency |
| Choreography | Local autonomy | Harder global reasoning and recovery |
| Rich workflow history | Replay and debugging | Storage, privacy, and migration burden |
| Minimal history plus case events | Cleaner business audit | Requires deliberate projection |
| Automatic assignment | Faster routing | Can ignore tacit team context |
| Pull queue | Operator autonomy | Cherry-picking and starvation |
| AI-generated summary | Lower review time | Omission, anchoring, hallucination |
| Strict revalidation | Prevents stale decisions | More rework after benign changes |
| Frequent escalation | Protects deadlines | Alert fatigue |
| Flexible manual repair | Faster recovery | Requires narrow authority and audit |
| Long retention | Investigation and compliance | Privacy and storage cost |

## Construction example: change-order and invoice exception case

A subcontractor invoice references a change order that appears approved in email, remains pending in the project system, and exceeds the ERP commitment. OCR also disagrees with one line-item total.

One idempotent intake command creates a case keyed by tenant, vendor, project, invoice number, and revision. The case snapshot links the invoice image, OCR claims, contract, commitment, change-order record, email approval, and drawing revision without copying broad project access.

The state machine moves from triage to evidence gathering. Parallel child workflows re-run deterministic totals, query the project system, request a field quantity confirmation, and check the ERP. The AI drafts a discrepancy summary with citations but cannot mark facts authoritative or place a hold.

The case enters a business-conflict exception. A project manager receives a task for quantity evidence, while accounting owns the financial reconciliation. The response SLA pauses only for an approved external-wait reason. A new change-order revision invalidates the old evidence packet.

After authorized approval, an execution adapter posts an ERP hold using an idempotency key. The ERP times out, so the case moves to outcome-unknown rather than retrying blindly. Reconciliation finds the hold, records the external ID, and completes the step. Closure requires the final disposition, linked ERP outcome, notifications, and a reason code.

Managers can see time spent waiting on field evidence, recurring ERP mismatches, AI proposals accepted or corrected, stale approvals, and cases reopened after payment disputes.

## Healthcare operations variation

A discharge-coordination case may involve clinical clearance, medication reconciliation, transportation, equipment, insurance authorization, and caregiver readiness. Each domain remains authoritative for its own decision.

The case can coordinate operational milestones and identify missing evidence, but AI must not make independent clinical decisions. Use strict PHI access, minimum necessary evidence packets, role-based tasks, clinical approval, patient-identity verification, emergency/manual paths, and audited communications. Track waiting reasons and access disparities without exposing sensitive details to unauthorized operations staff.

## Interview prompt

**Design a human-AI case-management platform for construction invoice and change-order exceptions. Cases may last weeks, collect changing evidence, cross project and ERP systems, require several roles, breach SLAs, merge with duplicates, and produce outcome-unknown external writes. Explain how you would adapt it for healthcare discharge operations.**

## Worked answer

1. **Define the case contract.** Identify outcome, owner, business key, lifecycle, deadlines, authority, evidence, and closure proof.
2. **Model explicit states.** Separate normal milestones, business exceptions, external waiting, execution, reconciliation, closure, and reopening.
3. **Persist versioned context.** Store case version, policy/workflow versions, evidence manifest, decisions, timers, child IDs, and operation ledger.
4. **Use durable orchestration.** Run retryable idempotent activities, durable timers, signals, cancellation, and version-safe child workflows.
5. **Design task operations.** Route by skill and authority; use leases, claim tokens, aging, protected capacity, and completion-time revalidation.
6. **Bound AI assistance.** Allow cited extraction, summaries, missing-evidence detection, and next-task proposals; keep state mutation and policy enforcement deterministic.
7. **Bind decisions to evidence.** Hash the evidence packet and action, expire approvals, and invalidate them after material change.
8. **Handle exceptions explicitly.** Classify validation, business, policy, dependency, security, capacity, uncertainty, and outcome-unknown cases with owners and remedies.
9. **Execute safely.** Revalidate state and authority, use idempotency keys and narrow credentials, and reconcile uncertain external outcomes before retry.
10. **Support lifecycle operations.** Authorize and audit reopen, merge, split, supersede, migrate, and manual repair without erasing history.
11. **Operate by outcomes.** Monitor state age, SLA, waiting reason, rework, reopen, override, external effects, AI quality, and closure outcomes.
12. **Adapt to healthcare.** Add PHI minimization, clinical authority, identity checks, approved urgency rules, access-equity metrics, and manual continuity.

**One-line conclusion:** A production human-AI workflow should make the business case—not the model session, task queue, or retry—the durable, versioned, accountable unit from intake through verified outcome.

## References

- [OMG Business Process Model and Notation 2.0.2](https://www.omg.org/spec/BPMN/2.0.2/)
- [OMG Case Management Model and Notation 1.1](https://www.omg.org/spec/CMMN/1.1/)
- [Temporal documentation — durable execution](https://docs.temporal.io/)
- [OpenTelemetry specification](https://opentelemetry.io/docs/specs/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [HL7 FHIR Workflow module](https://hl7.org/fhir/workflow.html)
