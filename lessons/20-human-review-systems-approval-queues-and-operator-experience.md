# 20 — Human review systems, approval queues, and operator experience

Previous: [Data contracts, schema evolution, and integration resilience](19-data-contracts-schema-evolution-and-integration-resilience.md). Study time: 50–60 minutes including the exercise.

## Mental model

Human review is a **scarce, failure-prone control plane**, not magic sprinkled onto an automated system. It should absorb consequential uncertainty, produce an accountable decision, and hand execution a decision that is still valid.

Model the path as:

```text
risk signal → routing policy → bounded queue → evidence packet → human decision
            → validity check → idempotent execution → outcome reconciliation
```

A review succeeds only when the right person sees the right evidence before the deadline, understands the decision, and authorizes the exact action that later executes. A click on “Approve” is not enough if inputs, policy, identity, or resource versions can change afterward.

> Route by consequence and uncertainty, design for reviewer attention, and bind approval to an immutable action plan that is revalidated at execution.

## When to require review

Use human review when one or more conditions apply:

- an action is financially, clinically, legally, or operationally consequential;
- model confidence is low **and calibrated for the relevant slice**;
- evidence is missing, contradictory, stale, or outside the model's known domain;
- a policy exception, threshold, or separation-of-duties rule applies;
- the system proposes an irreversible or hard-to-recover external action;
- the resource changed after the proposal was created;
- random sampling is needed to estimate automation quality;
- a new model, prompt, customer workflow, or integration is being validated.

Do not route everything to humans. Universal review creates delay, fatigue, rubber-stamping, and an unbounded hidden backlog. Deterministic validation, safe defaults, automatic rejection, or reversible automation may be better controls.

Confidence alone is not a routing policy. Combine:

```text
expected risk = probability of error × consequence of error
```

Then add policy requirements, evidence quality, novelty, user intent, and available reviewer capacity.

## Risk-based routing

A routing service should produce a reasoned decision:

| Input | Example |
| --- | --- |
| Consequence | payment amount, patient safety, schedule impact |
| Uncertainty | calibrated confidence, conflicting sources |
| Evidence quality | OCR quality, missing page, stale record |
| Policy | approval threshold, licensed-role requirement |
| Novelty | new vendor, unseen document type, drifted slice |
| Resource state | version changed, approval revoked |
| Capacity | queue age, qualified reviewer availability |

Possible routes:

- auto-accept a low-risk, reversible proposal;
- auto-reject a policy violation;
- request missing information;
- send to a standard review queue;
- require two independent reviewers;
- escalate to a specialist;
- hold execution until an external system is reconciled.

Record the policy version and routing features. Otherwise, later audits cannot explain why one case was automated and another was reviewed.

## Queue design

A review queue is a scheduling system. Store each case durably with:

- tenant and resource identity;
- case type and risk tier;
- created time, deadline, and escalation time;
- required reviewer role, project scope, and separation-of-duties constraints;
- proposal ID, immutable plan hash, input/resource versions, and policy version;
- evidence manifest and source references;
- status, assignment lease, attempt count, and version;
- decision, reason code, notes, reviewer identity, and timestamps;
- execution operation ID and reconciled outcome.

Prioritize with explicit policy, not a single FIFO list. A practical score can combine risk, deadline proximity, age, and fairness:

```text
priority = risk weight + deadline urgency + aging credit + fairness credit
```

Aging prevents low-priority work from starving. Per-tenant quotas prevent one customer’s backfill from consuming every reviewer. Emergency work may preempt routine work, but preemption rules must be visible and measurable.

## Capacity and service levels

Little’s Law still applies:

```text
work in progress = arrival rate × average review time
```

If 600 cases arrive per hour and a completed review averages 3 minutes of active time, the workload needs 30 continuously productive reviewer-hours per hour before breaks, meetings, rework, or burst headroom. A queue cannot compensate for a permanently overloaded workforce.

Forecast by skill and time window, not only total count. Measure:

- arrival and completion rates;
- oldest case and queue-age percentiles;
- active handling time and end-to-end decision time;
- abandonment, reassignment, and escalation;
- backlog drain time at current capacity;
- reviewer utilization without targeting 100%;
- service-level attainment by risk tier and tenant;
- decision quality and downstream correction, not clicks per hour.

At sustained overload, shed low-value sampling, pause non-urgent automation, request more reviewers, or fail closed for consequential actions. Never silently lower the review standard.

## Claiming, leases, and concurrency

Two reviewers must not unknowingly decide the same mutable case.

Use an atomic claim with a short renewable lease:

1. reviewer requests the next eligible case;
2. server verifies identity, role, tenant scope, and separation-of-duties;
3. compare-and-set changes `READY(version=8)` to `CLAIMED(version=9)`;
4. lease expires if the client disappears;
5. submission requires case version, lease token, and decision idempotency key;
6. server accepts only one terminal transition.

For collaborative review, model it deliberately: independent votes, quorum, adjudication, and conflict rules. Do not simulate collaboration through last-write-wins updates.

Offline review is risky because evidence and authorization may become stale. If supported, use short-lived signed case bundles, prohibit high-risk execution offline, and revalidate every dependency on reconnect.

## The evidence packet

The reviewer interface should answer five questions quickly:

1. What exact decision is required?
2. What action will happen if I approve?
3. What evidence supports or contradicts it?
4. What is uncertain or missing?
5. What changed since the proposal was generated?

Show:

- a plain-language action summary and structured parameters;
- source excerpts with page, field, and version citations;
- validation results and policy reasons;
- calibrated confidence plus slice warnings—not decorative percentages;
- differences from the current authoritative record;
- prior decisions, conflicts, and external outcome state;
- the deadline and consequence of delay;
- a preview of downstream side effects.

Use progressive disclosure so critical facts are visible and raw detail remains available. Preserve keyboard navigation, readable contrast, accessible labels, and safe focus order. Avoid default-selected approval, confusing button placement, or urgency styling that pressures acceptance.

Bulk actions are appropriate only for homogeneous, low-risk cases with visible selection criteria, per-item validation, a bounded batch size, a preview, and an auditable undo or recovery path.

## Approval-bound execution

Approval must authorize an exact plan, not a vague intention. Hash or otherwise bind:

- action type and normalized arguments;
- tenant and target resource IDs;
- expected resource and source versions;
- evidence manifest/version;
- policy and tool schema versions;
- approver identity and required roles;
- amount, currency, destination, or clinical parameters;
- expiry and maximum execution attempts.

Before executing, the worker rechecks authorization, separation of duties, plan hash, expiry, expected versions, and policy. If anything material changed, mark the approval `STALE` and return to review.

Execution uses a stable operation ID and idempotency key. A timeout after an external write becomes `OUTCOME_UNKNOWN`, not “failed.” Reconciliation queries the external system before any retry capable of duplicating the action.

A useful state machine is:

```text
DRAFT → READY → CLAIMED → APPROVED / REJECTED / NEEDS_INFO
                           ↓
                    EXECUTING → SUCCEEDED
                           ↘ OUTCOME_UNKNOWN → RECONCILED
APPROVED → STALE when bound inputs, policy, or authorization change
```

Terminal decisions are append-only audit events; corrections create a new decision version.

## Separation of duties and escalation

Policy may require:

- proposer cannot approve;
- document extractor cannot be the sole verifier;
- approvals above a threshold need two roles;
- a clinician, licensed professional, or project controller must decide;
- a reviewer cannot approve outside assigned tenant/project scope.

Escalation should identify a responsible role, deadline, fallback, and notification policy. “Send to admin” is not an escalation design.

Define what happens when no qualified reviewer is available. For high-consequence actions, safe behavior is usually to hold and communicate delay. For time-critical healthcare operations, escalation may go to an on-call clinical role under an approved downtime procedure; automation must not invent authority.

## Reviewer quality and disagreement

Reviewer actions are labels only after quality checks. People make mistakes, interpret policy differently, and adapt behavior to productivity metrics.

Use:

- gold or adjudicated cases sampled carefully;
- periodic double review to estimate agreement;
- reason codes plus optional notes;
- reviewer calibration and policy training;
- disagreement analysis by case type and evidence quality;
- appeals and corrections;
- downstream outcome audits;
- drift monitoring after policy or UI changes.

Agreement is not truth: reviewers can consistently share the same mistake. Compare against adjudication, later authoritative outcomes, and business harm.

Do not train a model directly on every approval. Reviewer decisions may encode access limitations, queue pressure, outdated policy, automation bias, or demographic and tenant-specific bias. Preserve lineage and select training labels under an explicit governance process.

## Automation bias and fatigue

Review interfaces can make humans defer to the model. Mitigations include:

- require reviewers to inspect primary evidence for high-risk decisions;
- reveal a model recommendation after an independent judgment for selected audits;
- show uncertainty and contradictory evidence, not persuasive explanations alone;
- rotate repetitive work and cap continuous sessions;
- insert breaks or secondary review when handling time collapses;
- monitor approval streaks and anomalously fast decisions as investigation signals;
- test the UI with seeded errors;
- reward correct escalation, not only throughput.

Do not punish reviewers for selecting “needs information.” If the metric optimizes speed alone, the queue will become a click factory.

## Construction example

An assistant extracts an invoice for $184,250 and proposes coding it to a cost account before sending it to the ERP.

1. Deterministic checks validate tenant, currency, arithmetic, purchase order, duplicate invoice key, and required lien-waiver evidence.
2. Routing assigns high risk because the amount exceeds the controller threshold and the change-order balance is uncertain.
3. The case packet shows invoice pages, extracted fields, purchase-order balance, change-order status, duplicate search, and the exact ERP request.
4. A project manager reviews coding; a controller independently approves payment. Neither can approve their own proposal.
5. The approval binds amount, currency, vendor, account, project, document version, purchase-order version, policy version, and ERP tool schema.
6. A new change order arrives before execution. The version check marks approval stale and explains the changed balance.
7. After reapproval, execution uses operation ID `invoice:{tenant}:{invoice-version}`.
8. The ERP times out. The workflow shows “confirmation pending,” prevents a blind retry, and reconciles by operation/reference ID.
9. Reviewer disagreement and later corrections feed evaluation analysis with full lineage, not automatic retraining.

## Healthcare operations variation

A discharge-planning assistant proposes a follow-up appointment and flags a medication reconciliation issue.

Routing considers clinical consequence, missing evidence, patient identity confidence, source freshness, and whether the reviewer holds the required clinical role. The packet separates scheduling facts from clinical recommendations, cites current medication and discharge records, and marks any record changed since generation.

A scheduler may confirm appointment logistics but cannot approve a medication decision. A clinician reviews the medication discrepancy. If the medication list changes, the prior clinical approval becomes stale. Time-critical escalation follows an approved clinical on-call path. The interface avoids exposing unnecessary protected health information, and audit logs reference restricted evidence rather than copying it.

The system supports clinical work; it does not convert an operational reviewer into a clinician or treat model confidence as medical authority.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Review every case | Maximum nominal oversight | Backlog, fatigue, rubber-stamping |
| Risk-based routing | Focuses scarce expertise | Calibration and policy complexity |
| Single reviewer | Faster and cheaper | Higher individual-error risk |
| Two-person approval | Separation of duties | Latency and staffing cost |
| FIFO | Simple and predictable | Urgent work waits |
| Priority plus aging | Meets risk deadlines | Tuning and starvation bugs |
| Rich evidence packet | Better decisions | Cognitive overload if poorly designed |
| Bulk approval | High throughput | Correlated mass error |
| Short lease | Recovers abandoned work | Interrupts long cases |
| Long lease | Reviewer continuity | Cases remain stranded |
| Immutable plan binding | Prevents stale execution | More re-review |
| Reviewer feedback for training | Domain-specific labels | Bias and leakage risk |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Every case requires review | Unbounded backlog | Risk tiers and safe automation |
| Confidence is uncalibrated | Wrong cases bypass review | Slice calibration and outcome audits |
| FIFO ignores deadlines | High-risk SLA breach | Risk/deadline priority with aging |
| No atomic claim | Duplicate decisions | Compare-and-set plus lease |
| Lease expires during submission | Lost or conflicting work | Idempotent decision and version check |
| Approval is not plan-bound | Different action executes | Hash exact plan and versions |
| Resource changes after approval | Stale harmful action | Revalidate and invalidate |
| Retry after external timeout | Duplicate payment/action | Outcome-unknown state and reconciliation |
| Default approval button | Automation bias | Neutral UI and evidence-first flow |
| Bulk approval spans mixed risk | Correlated error | Homogeneous bounded batches |
| Throughput-only incentives | Rubber-stamping | Quality and outcome metrics |
| Reviewer label treated as truth | Biased training data | Adjudication and lineage |
| Missing qualified reviewer | Unauthorized decision | Role-aware escalation or hold |
| Raw sensitive evidence in logs | Privacy incident | Restricted references and redaction |
| Queue hidden from operations | Silent SLA failure | Age, capacity, and drain-time alerts |

## What to measure

Track routing volume and reasons, auto-action rate, review rate, false accept/reject estimates, overrides, escalations, disagreement and adjudication, correction rate, downstream harm, case arrival/completion, backlog age and drain time, SLA by tier, handling time distribution, abandoned leases, stale approvals, authorization failures, execution outcomes, outcome-unknown age, reviewer workload, and performance by meaningful slice.

Pair system metrics with user outcomes. Faster review is not an improvement if corrections, missed evidence, or adverse outcomes increase.

## Interview prompt

> Design the human review and approval subsystem for a multi-tenant operations assistant. It extracts construction invoices, proposes ERP actions, and supports healthcare discharge workflows. Reviews have deadlines, different role requirements, bursty volume, model uncertainty, changing source records, and external systems that can time out after accepting writes.

Cover routing, prioritization, capacity, evidence UX, claiming/concurrency, authorization, separation of duties, approval binding, stale inputs, idempotent execution, reconciliation, feedback quality, privacy, observability, and overload behavior.

## Worked answer

**One-line conclusion:** Treat human review as a risk-routed, capacity-bounded workflow whose decisions authorize immutable, versioned plans that are revalidated before idempotent execution.

“I would make routing a policy service using consequence, calibrated uncertainty, evidence quality, novelty, and mandatory role rules. Cases enter durable queues partitioned by skill and tenant, prioritized by risk and deadline with aging and fairness. Each case carries an evidence manifest, exact action preview, resource versions, policy version, and plan hash. An atomic compare-and-set claim issues a renewable lease; decision submission is idempotent and version-checked. The interface foregrounds contradictory or missing evidence, uses neutral controls, and restricts bulk approval to homogeneous low-risk cases.

Authorization and separation of duties are checked at assignment, decision, and execution. An approval binds action arguments, tenant/resource IDs, source versions, tool schema, policy, roles, and expiry. The worker revalidates those fields; changed inputs create a stale approval rather than executing. External writes use stable operation IDs. Timeouts enter outcome-unknown and reconcile before retry. Capacity dashboards show arrival, completion, queue age, drain time, and SLA by risk tier; overload pauses low-value sampling and holds high-risk actions safely. Reviewer disagreement, corrections, and downstream outcomes are adjudicated and versioned before use in evaluation or training. Construction queues require project and controller roles; healthcare queues enforce clinical boundaries and minimum necessary data.”

## Practice lab

Design a review system where:

- invoice extraction confidence is 0.93 but poorly calibrated for scanned change orders;
- invoices above $100,000 require a controller and proposer/approver separation;
- two reviewers claim the same case during a network partition;
- a change order updates the available balance after approval;
- the queue receives a 4× month-end burst;
- the ERP times out after a possible accepted write;
- reviewer actions may later become training labels;
- healthcare cases share the platform but require clinical roles and stricter data minimization.

Specify routing features, case state, queue priority, leases, evidence packet, approval-bound fields, stale-decision behavior, capacity and overload policy, ERP reconciliation, reviewer-quality controls, metrics, and audit events.

## References

- [NIST — AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [NIST — AI RMF Generative AI Profile](https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence)
- [Amershi et al. — Guidelines for Human-AI Interaction](https://www.microsoft.com/en-us/research/publication/guidelines-for-human-ai-interaction/)
- [FDA — Applying Human Factors and Usability Engineering to Medical Devices](https://www.fda.gov/regulatory-information/search-fda-guidance-documents/applying-human-factors-and-usability-engineering-medical-devices)
- [Google SRE Book — Handling Overload](https://sre.google/sre-book/handling-overload/)
