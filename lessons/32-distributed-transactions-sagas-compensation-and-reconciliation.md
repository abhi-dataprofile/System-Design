# 32 — Distributed transactions, sagas, compensation, and reconciliation

Previous: [Time, ordering, logical clocks, and causal consistency](31-time-ordering-logical-clocks-and-causal-consistency.md). Study time: 55–65 minutes including the exercise.

## Mental model

A local database transaction can make one resource change atomically. A business workflow across databases, queues, humans, and external APIs cannot usually be rolled back as if nothing happened.

> Keep hard invariants inside the smallest authoritative transaction; coordinate everything else as durable, idempotent steps with explicit recovery.

Separate four mechanisms:

| Mechanism | Purpose | Important limit |
| --- | --- | --- |
| Local ACID transaction | Atomic change within one transaction manager | Cannot include arbitrary external effects safely |
| Distributed transaction/2PC | Atomic commit across participating transactional resources | Blocking, coordination, limited participant support |
| Saga | Sequence of durable local transactions | Intermediate states are visible |
| Compensation | Business action that mitigates a completed step | Usually not a true inverse |
| Reconciliation | Compare recorded intent with actual outcome and repair/escalate | May require external evidence or humans |

A refund does not erase a charge. A cancellation does not make a notification unread. An ERP reversal creates another accounting event. Call these compensations, not rollbacks.

## When to use each approach

Use a **local transaction** for invariants owned by one database:

- one invoice approval transition;
- unique operation ID;
- ledger entry plus outbox event;
- budget reservation row;
- workflow state and next-step command.

Use **two-phase commit** only when all resources support the protocol, strong atomic visibility is required, latency and availability costs are acceptable, and the operational team can handle in-doubt transactions. It rarely works with SaaS APIs, email, object storage, humans, or model providers.

Use a **saga** when:

- work spans multiple service-owned stores or external systems;
- steps may take seconds to days;
- human approval is involved;
- each step can be retried idempotently;
- intermediate states can be modeled explicitly;
- failures have a meaningful compensation, forward recovery, or manual path.

Use **reconciliation** whenever an external outcome can be unknown, systems can drift, or compensation can fail.

Do not split one invariant across services merely to justify a saga. If two rows must always change together and share one lifecycle, co-locate them under one transaction boundary when practical.

## Start with the business invariant

Before drawing services, state what must never be false.

For invoice execution:

- one stable operation represents one approved intent;
- execution refers to the exact approved invoice/evidence version;
- the system never blindly repeats an outcome-unknown ERP write;
- budget and accounting states remain explainable;
- every terminal state has evidence;
- tenant authorization is revalidated at consequential boundaries.

Then classify state:

- **authoritative:** invoice version, approval, operation ledger;
- **reserved:** budget or capacity held temporarily;
- **derived:** search, analytics, notification views;
- **external:** ERP posting, payment, email;
- **observed:** last external status and reconciliation evidence.

The saga coordinates these states; it does not make all of them one ACID transaction.

## Local transaction plus transactional outbox

A common failure is updating the database and publishing a message separately:

1. database commit succeeds;
2. process crashes before publish;
3. downstream never learns about the change.

Or the message publishes first and the database rolls back.

Store the domain change and outbox record in one local transaction:

```sql
BEGIN;

UPDATE invoice_operations
SET state = 'APPROVED',
    version = version + 1
WHERE operation_id = :operation_id
  AND state = 'PENDING_REVIEW'
  AND input_version = :approved_version;

INSERT INTO outbox (
  event_id, aggregate_id, aggregate_version, event_type, payload
) VALUES (...);

COMMIT;
```

A relay publishes unsent outbox entries. It may publish more than once, so consumers need an inbox/deduplication key and idempotent state transition.

Marking an outbox row “published” is delivery bookkeeping, not proof every consumer applied it. Consumer lag and reconciliation remain visible.

## Saga state machine

Model the workflow explicitly:

```text
PENDING_REVIEW
  → APPROVED
  → BUDGET_RESERVED
  → ERP_SUBMITTING
  → ERP_CONFIRMED
  → COMPLETED

Failure branches:
  → COMPENSATING
  → COMPENSATED
  → OUTCOME_UNKNOWN
  → MANUAL_REVIEW
  → FAILED_FINAL
```

Each transition records:

- workflow and operation ID;
- current state and version;
- input/reference versions;
- step attempt;
- idempotency key;
- deadline;
- actor/policy/configuration version;
- external request/reference;
- result or unknown outcome;
- next retry time;
- compensation state;
- audit metadata.

Persist state before and after external calls so recovery can determine what may have happened. Never encode the only workflow truth in process memory.

## Orchestration versus choreography

### Orchestration

A workflow coordinator commands each step and records progress.

Benefits:

- explicit state machine;
- central visibility and timeout handling;
- easier compensation order;
- good fit for long-running human and external workflows.

Costs:

- coordinator becomes important infrastructure;
- can accumulate business coupling;
- must scale and recover durably.

The orchestrator owns coordination, not every domain invariant. Each service validates its command and owns its local transaction.

### Choreography

Services react to events and emit new events.

Benefits:

- loose direct coupling;
- natural for simple, acyclic propagation;
- independent service evolution.

Costs:

- end-to-end state is harder to see;
- event loops and hidden dependencies;
- compensation becomes scattered;
- debugging and change impact are difficult.

Use choreography for simple derived reactions such as analytics and notification projections. Prefer orchestration for consequential, multi-step workflows with deadlines, approval, and compensation.

Avoid mixing both without an ownership rule. Define which component declares the workflow terminal.

## Two-phase commit

Two-phase commit has:

1. **prepare:** coordinator asks participants to durably promise they can commit;
2. **commit/abort:** coordinator records and announces the decision.

It can provide atomic commitment across compatible resources. Costs include:

- locks/resources held during coordination;
- coordinator and participant recovery;
- in-doubt states during failures;
- latency across failure domains;
- reduced availability under partitions;
- poor fit for long human steps;
- no support from most external APIs.

A transaction manager can recover a recorded decision, but operators still need visibility into prepared transactions and blocked resources.

Do not label “call two APIs, then undo one if the second fails” as 2PC. That is a saga with compensation.

## Try-confirm-cancel and reservations

When a resource supports reservations:

1. **Try:** reserve budget, inventory, capacity, or appointment slot;
2. **Confirm:** finalize after all preconditions hold;
3. **Cancel:** release before expiry if the workflow fails.

Reservations need:

- stable reservation ID;
- owner/workflow ID;
- amount/resource scope;
- expiration;
- idempotent confirm/cancel;
- conditional state transitions;
- reconciliation of expired or orphaned holds.

A reservation reduces oversubscription but adds temporarily unavailable capacity. Expiration is not a complete cleanup guarantee; a sweeper and reconciliation should verify release.

Do not reserve an external irreversible effect. “Reserve an email” has no useful meaning; defer sending instead.

## Compensation design

For every forward step, ask:

- Can it be retried?
- Can it be canceled before completion?
- Is there a business inverse afterward?
- Who authorizes compensation?
- What if compensation fails?
- What evidence confirms it?
- Does compensation create a new financial, legal, or clinical event?

Examples:

| Forward step | Possible compensation | Limitation |
| --- | --- | --- |
| Reserve budget | Release reservation | Only before final posting |
| Create ERP draft | Void draft | Vendor rules may restrict |
| Post invoice | Reversal/credit entry | New accounting record, approval may be required |
| Send notification | Send correction | Original remains read |
| Publish search document | Tombstone/new version | Caches may retain old view briefly |
| Assign reviewer | Release claim | Human may already have acted |
| Book appointment | Cancel/rebook | Patient impact and policy apply |

Compensations must be idempotent and stateful. Record `compensation_requested`, `compensating`, `compensated`, or `compensation_failed`. Do not repeatedly issue a reversal because the first response timed out.

Compensate in dependency-aware reverse order when appropriate, but business rules may require a different sequence.

## Forward recovery versus compensation

Sometimes the correct response is to finish the workflow rather than undo it.

Use forward recovery when:

- the external effect is already valid and irreversible;
- compensation would cause more harm;
- remaining steps can safely resume;
- policy requires completion;
- users expect the committed outcome.

Example: ERP posting succeeded, but analytics publication failed. Do not reverse the ERP invoice just to restore technical symmetry. Repair the derived projection.

Classify each step as:

- must compensate;
- should complete forward;
- optional/best effort;
- requires human decision.

## Outcome unknown

A timeout means the caller stopped waiting, not that the dependency did nothing.

For an external write:

1. record `SUBMITTING` with operation/idempotency key before the call;
2. send the request;
3. on confirmed success, store external reference and `CONFIRMED`;
4. on explicit safe failure, decide retry or compensation;
5. on timeout/disconnect, store `OUTCOME_UNKNOWN`;
6. reconcile by idempotency key, external reference, or domain query;
7. retry only after evidence says no effect occurred;
8. escalate if the provider cannot prove either outcome.

Reserve rate-limit capacity for reconciliation. Unknown outcomes are high-priority correctness work, not ordinary failed jobs.

If the external system supports neither idempotency nor lookup, strict exactly-once behavior is impossible. Reduce risk with a controlled adapter, serialized operation ledger, unique business reference, delayed operator verification, and explicit residual-risk policy.

## Isolation anomalies across saga steps

Sagas expose intermediate state. Other workflows may observe or modify resources between steps.

Mitigations:

- semantic lock/state such as `invoice_execution_pending`;
- reservation;
- version check before every step;
- commutative operation;
- invariant revalidation;
- escrowed capacity;
- compensating event;
- bounded workflow duration.

A semantic lock is domain state, not a process mutex. It should have ownership, version, expiration/recovery policy, and user-visible meaning.

Do not hold a database transaction open while waiting for a human or remote API.

## Idempotency and deduplication

Use distinct identifiers:

- **workflow ID:** one saga instance;
- **operation ID:** one business intent;
- **step ID:** one logical step;
- **attempt ID:** one execution attempt;
- **event ID:** one message;
- **external idempotency key:** provider dedupe scope.

A retry creates a new attempt ID but reuses the operation and step idempotency key. Otherwise the provider sees a new action.

Idempotency records need request fingerprint, status, result/reference, owner, and retention at least as long as retries/replays are possible. Reject reuse of one key with a different payload.

Consumer deduplication must be atomic with the consumer’s local state change. “Check then act” in two transactions has a race.

## Deadlines, retries, and human steps

A saga has several time budgets:

- user-response deadline;
- per-step timeout;
- retry window;
- approval expiration;
- reservation expiration;
- business completion target;
- reconciliation escalation age.

Do not let a technical retry extend business authorization indefinitely. At execution, revalidate approval, actor authorization, input versions, policy, and reservation.

Human review may take days. Persist the workflow, notify without treating notification delivery as approval, and handle role revocation or changed evidence.

## Reconciliation architecture

Reconciliation compares at least three views:

```text
internal intent
vs internal recorded state
vs external observed state
```

A reconciler should:

1. select overdue/nonterminal/unknown operations;
2. fetch authoritative internal versions;
3. query external status using stable references;
4. classify: matched, missing, duplicated, divergent, inaccessible;
5. apply an idempotent repair or create an operator case;
6. record evidence and next action;
7. close only when terminal criteria are proven.

Use both continuous reconciliation for urgent cases and periodic sweeps for silent drift. Partition and rate-limit sweeps so they do not overload dependencies.

Dashboards need counts and age distributions, not only a total backlog. A single high-value unknown outcome may be more important than thousands of delayed analytics events.

## Cross-service observability

Trace by workflow, operation, and step—not only request ID.

Record:

- state transition latency;
- attempts and retry reasons;
- outbox age and publish count;
- inbox duplicates;
- reservation age;
- outcome-unknown age;
- compensation requested/completed/failed;
- external references;
- reconciliation classifications;
- manual-review queue age;
- terminal business outcome;
- model/policy/input versions for AI-assisted decisions.

Logs should not contain tenant secrets, invoice documents, PHI, or unrestricted model prompts.

## AI agents and sagas

An LLM should not be the saga coordinator or source of truth.

The model may:

- extract candidate fields;
- recommend a next step;
- summarize evidence;
- classify an exception;
- draft an operator explanation.

Deterministic code must:

- validate structured output;
- authorize tools;
- enforce the workflow state machine;
- bind approvals to exact inputs;
- generate/reuse idempotency keys;
- persist before side effects;
- decide retries and compensation;
- reconcile outcomes.

A tool call proposed twice by a model must still map to one operation ID. Model “confidence” does not decide whether an unknown payment or ERP posting is retried.

## Construction operations example

A construction platform approves and posts a subcontractor invoice:

1. ingest and validate invoice version 8;
2. confirm matching contract/change-order evidence;
3. obtain superintendent and project-manager approval;
4. reserve remaining commitment budget;
5. create/post the ERP invoice;
6. publish job-cost projection;
7. notify accounting and the project team.

Design:

- invoice, approval binding, operation ledger, and outbox commit locally;
- the durable orchestrator commands steps;
- budget reservation has a stable ID and expiry;
- ERP command uses the operation ID as idempotency/business reference;
- before ERP execution, revalidate invoice/evidence versions, policy, roles, and reservation;
- ERP timeout becomes `OUTCOME_UNKNOWN`;
- reconciliation queries by operation/business reference;
- if ERP did not post, retry within approval validity;
- if it posted, continue forward to projections and notification;
- if budget reservation expired before posting, reacquire and revalidate;
- if posting must be reversed, create an approved reversal—not a hidden deletion;
- projection and notification failures recover forward;
- notification compensation is a correction message;
- every terminal state has external and internal evidence.

If a change order modifies the invoice after approval but before execution, the expected-version transition fails. The workflow becomes `STALE_INPUT`, releases the reservation, and requests review against the new version.

## Healthcare operations variation

A healthcare operations workflow coordinates scheduling, eligibility checks, prior authorization, reminders, and operational AI assistance.

Patient/clinical source systems retain authority. The saga records exact resource versions and authorization context. A scheduling hold can be reserved and canceled; a sent reminder can only be corrected; a submitted prior-authorization request may require status lookup rather than blind resubmission.

Clinical or patient-impacting compensation follows approved policy and may require human review. A technical failure never silently reverses a clinical decision. PHI is minimized in events and workflow metadata; access is rechecked at each consequential step.

If an interface timeout leaves submission status unknown, reconciliation queries the authoritative system and surfaces unresolved cases. Model suggestions can help summarize evidence but cannot declare a compensation successful.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| One local transaction | Strong, simple invariant | Limited resource boundary |
| Two-phase commit | Atomic multi-resource decision | Blocking and limited integrations |
| Orchestrated saga | Visible control and recovery | Coordinator complexity |
| Choreography | Loose direct coupling | Hidden workflow and debugging cost |
| Reservation/TCC | Prevents oversubscription | Held capacity and expiry cleanup |
| Compensation | Business recovery | Not true rollback |
| Forward recovery | Preserves valid effects | Derived inconsistency until repaired |
| Transactional outbox | No lost DB-to-event handoff | At-least-once publish |
| Inbox dedupe | Safe repeated delivery | State and retention cost |
| Semantic lock | Protects long business step | Stale ownership/recovery rules |
| Automatic compensation | Fast repair | Can amplify harm if policy is wrong |
| Human exception queue | Handles ambiguity | Cost and delay |
| Continuous reconciliation | Fast drift detection | Dependency load |
| Periodic sweep | Finds silent failures | Longer inconsistency window |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Database commit then publish crash | Lost event | Transactional outbox |
| Message publish then DB rollback | Phantom event | Commit state and outbox together |
| Consumer check then act race | Duplicate effect | Atomic inbox plus state update |
| Process memory owns saga | Progress lost on crash | Durable state machine |
| Open DB transaction waits for human | Locks and outage | Commit local state; resume later |
| Compensation assumed inverse | Audit/accounting harm | Model real business action |
| Compensation response times out | Duplicate reversal | Compensation idempotency/reconciliation |
| Unknown external result marked failed | Blind duplicate retry | Explicit outcome-unknown state |
| New idempotency key per retry | Repeated external action | Stable operation/step key |
| Approval outlives changed input | Wrong version executes | Bind and revalidate versions |
| Reservation expires unnoticed | Invalid confirm | Conditional confirm and sweeper |
| Choreography forms event loop | Repeated work | Ownership and transition guards |
| Orchestrator bypasses service invariant | Corrupt local state | Service validates commands |
| Retry extends expired authorization | Unauthorized side effect | Business deadline revalidation |
| Analytics failure reverses ERP | Unnecessary financial action | Forward repair derived state |
| Reconciler runs without rate limit | Dependency overload | Prioritized bounded sweeps |
| Manual case lacks evidence | Unsafe operator decision | Complete operation timeline |
| LLM directly retries tool | Duplicate consequential action | Deterministic operation ledger |

## What to measure

Track workflows by state and age; step latency; retry and exhaustion rate; outbox backlog/age; publish duplicates; inbox dedupe rate; reservation utilization and expiry; outcome-unknown count and age; compensation success/failure/unknown; forward-recovery backlog; reconciliation matches, missing, duplicates, divergences, and access failures; manual-review age; stale-approval rejection; external idempotency conflicts; and terminal business outcomes.

Alert on any consequential unknown beyond its objective, growing outbox/reconciliation lag, repeated compensation, orphaned reservations, or a workflow with no durable progress.

## Interview prompt

> Design a distributed invoice-approval and ERP-posting workflow for a multi-tenant construction AI platform. Approval, budget, document, notification, and ERP systems have separate boundaries. Humans may take days, evidence can change after approval, messages are delivered at least once, ERP calls may time out after succeeding, and ERP reversal requires separate approval.

Cover transaction boundaries, outbox/inbox, saga style, reservations, state machine, idempotency, compensation versus forward recovery, outcome reconciliation, AI/tool safety, observability, and healthcare adaptation.

## Worked answer

**One-line conclusion:** Commit each invariant locally, coordinate durable idempotent saga steps, treat compensation as a new governed business action, and reconcile every outcome that cannot be proven.

“I would keep the invoice version, approval binding, operation ledger, and outbox in one authoritative database transaction. The outbox relay publishes at least once; consumers atomically record event IDs with their local state changes.

A durable orchestrator owns an explicit saga state machine. It reserves budget using a stable reservation ID, then revalidates approval, authorization, policy, and evidence versions before compare-and-swapping into ERP submission. Every retry reuses the step’s idempotency key while getting a new attempt ID.

If the ERP confirms success, I store its reference and complete remaining projections and notifications through forward recovery. If it explicitly rejects safely, I release the reservation or route for correction. If the call times out, the state is outcome unknown; reconciliation queries by operation/business reference before any retry. If the ERP cannot deduplicate or query, I expose the residual risk and require operator verification.

A changed invoice version makes the approval stale and prevents execution. A reversal is not rollback: it is a separately authorized accounting operation with its own ID, state, and reconciliation. A notification can only be corrected, while derived search or analytics state can be rebuilt.

The model can extract fields or propose actions, but deterministic code validates outputs, enforces transitions, persists state, controls tools, and decides retry/compensation policy. I would monitor state age, outbox lag, duplicates, reservation expiry, unknown outcomes, compensation failure, reconciliation classifications, and manual-review age.”

## Practice lab

An invoice is approved and its budget is reserved. The evidence changes just before execution. Two duplicate approval events arrive. The ERP accepts the invoice but the response is lost. The budget reservation expires, the notification was already sent, and the ERP supports lookup but reversal requires accounting approval.

Specify:

- the local transaction boundary;
- outbox and inbox records;
- saga states and versions;
- operation, step, attempt, and event IDs;
- reservation behavior;
- approval invalidation;
- outcome-unknown reconciliation;
- forward recovery versus compensation;
- approved reversal workflow;
- notification correction;
- AI tool boundary;
- metrics and operator evidence.

Then adapt the design to scheduling and prior authorization in healthcare.

## References

- [Pat Helland — Life beyond Distributed Transactions](https://queue.acm.org/detail.cfm?id=3025012)
- [Chris Richardson — Saga pattern](https://microservices.io/patterns/data/saga.html)
- [AWS Prescriptive Guidance — Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [Azure Architecture Center — Compensating Transaction pattern](https://learn.microsoft.com/azure/architecture/patterns/compensating-transaction)
- [Temporal — Durable execution and failure handling](https://docs.temporal.io/)
