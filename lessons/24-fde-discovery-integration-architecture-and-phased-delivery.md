# 24 — FDE discovery, integration architecture, and phased delivery

Previous: [Batch processing, stream processing, and change data capture](23-batch-stream-processing-and-change-data-capture.md). Study time: 55–65 minutes including the exercise.

## Mental model

A Forward Deployed Engineer does not begin with architecture. The job begins with an operational outcome, the people responsible for it, and the evidence showing why the current workflow fails.

```text
business outcome
  → current workflow and failure evidence
  → constraints and source-of-truth map
  → smallest useful intervention
  → measurable pilot
  → hardened integration
  → repeatable product capability
```

Discovery and delivery are one feedback system. Every production increment should reduce a named risk, test an assumption, or improve a measurable user outcome.

> Start from the decision or action that must improve, trace its data and failure path end to end, then deliver the smallest reversible slice that proves value without hiding operational risk.

An impressive demo is not a successful deployment. A successful deployment produces a trusted outcome, fits human ownership, survives partial failure, and can be operated after the FDE leaves.

## When to use this approach

Use discovery-led phased delivery when:

- the customer describes symptoms rather than a precise requirement;
- workflows cross email, spreadsheets, SaaS tools, databases, and human approvals;
- data quality and API capability are uncertain;
- the environment is regulated, safety-sensitive, or operationally consequential;
- an AI solution’s usefulness depends on workflow adoption, not only model accuracy;
- different stakeholders disagree on the current process or source of truth;
- implementation must show value before a broad platform investment;
- the work may become a reusable product capability for other tenants.

Do not run months of discovery without testing anything. Use short, evidence-producing loops. Conversely, do not treat a stakeholder’s first requested feature as the validated problem.

## Discovery starts with outcomes

Translate “we need AI” into an operational statement:

```text
For [actor], when [trigger], help them [decision/action]
using [authoritative evidence], within [time],
while respecting [risk/approval constraints],
and improve [metric] from [baseline] to [target].
```

Weak requirement:

> Build an agent that reads construction documents.

Stronger requirement:

> When a project manager receives an invoice, assemble cited purchase-order, change-order, and duplicate-payment evidence within five minutes; propose coding but require authorized approval before any ERP write.

Ask what happens if the system is wrong, late, or unavailable. That answer shapes the architecture more than a feature list.

## Stakeholder and ownership map

Identify:

| Role | What to learn |
| --- | --- |
| Executive sponsor | Outcome, budget, priority, acceptable risk |
| Workflow owner | Current policy, exceptions, definition of done |
| Frontline operator | Actual steps, workarounds, cognitive load |
| IT/security | Identity, network, data, deployment constraints |
| Data/system owner | Source semantics, APIs, quotas, change process |
| Compliance/legal | Retention, purpose, required review, audit |
| Support/on-call | Failure triage, escalation, recovery |
| End customer/patient/vendor | Experience, consent, communication impact |

The loudest stakeholder is not necessarily the workflow owner. Record a RACI or equivalent decision map: who recommends, approves, implements, operates, and is informed.

Find the person who owns the business exception. Happy paths are usually documented; exceptions reveal the real system.

## Observe the current workflow

Use interviews, screen sharing, artifact review, and workflow shadowing. Build a service blueprint containing:

- trigger and entry channel;
- actors and handoffs;
- systems touched;
- data copied or transformed;
- decision rules;
- wait time versus active work;
- exceptions, rework, and escalations;
- authoritative source at each step;
- irreversible actions;
- current metrics and missing instrumentation.

Distinguish stated process from observed process. A policy may say approvals happen in the ERP while operators actually resolve missing cost codes through email and spreadsheets.

Do not record sensitive data casually during discovery. Use approved notes, redacted examples, and least-privilege access.

## Quantify the baseline

A useful baseline measures distributions and failure categories, not only averages:

- volume by day, tenant, location, and case type;
- active handling time and total cycle time;
- queue age and deadline misses;
- rework, exception, and escalation rate;
- error cost or operational consequence;
- integration failure and reconciliation rate;
- user adoption and workaround rate;
- model/data quality by meaningful slice;
- current labor and vendor cost.

If instrumentation does not exist, create a short manual sample with documented selection limits. Never invent ROI from anecdotes.

A value hypothesis can be:

```text
annual benefit =
eligible volume × adoption × improvement per completed case
− operating cost − implementation/change cost − expected error cost
```

State uncertainty ranges. “Could save three hours per day” is a hypothesis until measured against actual eligible work and sustained adoption.

## Requirement hierarchy

Separate requirements into:

1. **business outcome:** what improves;
2. **workflow:** who decides and acts;
3. **functional:** what the system does;
4. **data:** sources, meaning, freshness, lineage;
5. **integration:** reads, writes, quotas, failure behavior;
6. **security/privacy:** identity, isolation, retention, approval;
7. **reliability:** SLO, degradation, reconciliation, recovery;
8. **operability:** ownership, alerts, support, cost;
9. **adoption:** training, feedback, change management;
10. **exit:** success, rollback, and stop criteria.

Rank each as must, should, could, or later. A “must” needs a reason tied to risk or outcome.

Write non-goals. They protect the pilot from becoming an invisible platform rewrite.

## Assumption and risk register

For each material assumption record:

- claim;
- evidence;
- confidence;
- impact if false;
- cheapest test;
- owner;
- due date;
- decision it blocks.

Examples:

| Assumption | Cheap test |
| --- | --- |
| ERP API exposes stable invoice IDs | Read-only API spike on 50 records |
| Project IDs align across systems | Sample join and mismatch report |
| Operators trust cited suggestions | Usability test with historical cases |
| Approval policy is consistent | Exception workshop with workflow owner |
| OCR is adequate on scanned invoices | Stratified evaluation by document quality |
| Write API supports idempotency | Sandbox timeout/retry experiment |
| Healthcare data may use model endpoint | Security/privacy architecture review |

Attack the highest-risk assumption early, not the easiest coding task.

## Source-of-truth and integration map

For every entity, name the authority:

| Entity | Authority example | Derived consumers |
| --- | --- | --- |
| Invoice | ERP | Review queue, analytics, search |
| Purchase order | ERP/project system | Validation and coding |
| RFI/change order | Project platform | Retrieval, risk signals |
| User/project role | Identity and project systems | Authorization policy |
| Document binary/version | Object/project system | OCR, chunks, embeddings |
| Patient encounter | Clinical system | Operational queue |
| Approval decision | Workflow service | ERP command and audit |

Draw reads, writes, callbacks, files, users, and trust boundaries. Record authentication method, API scopes, rate limits, pagination, webhooks, consistency, versioning, sandbox quality, and escalation contact.

“Integration available” is not binary. A vendor may allow reads but not writes, omit historical changes, return eventual status, or provide no idempotency.

## Choose the intervention boundary

There are four common options:

- **system of record replacement:** highest control and migration risk;
- **workflow orchestration around existing systems:** often best for cross-system coordination;
- **decision support:** read-only cited recommendations with humans acting;
- **analytics/visibility:** derived views and alerts without operational control.

Start at the least invasive boundary that can prove the outcome. Read-only decision support may validate data and trust before write automation. Avoid duplicating the entire ERP or clinical record.

The system should clearly state which data is authoritative, cached, inferred, or awaiting reconciliation.

## Build, buy, configure, or integrate

Evaluate:

- strategic differentiation;
- time to first value;
- available product/API capability;
- security and deployment fit;
- total operating cost;
- customization depth;
- vendor lock-in and exit path;
- reliability ownership;
- data portability;
- support maturity.

A custom model is rarely the first differentiator. Workflow integration, proprietary context, evaluation, and operator experience may create more value.

Prefer a thin vertical slice across real boundaries over a broad mock interface that postpones every difficult integration.

## Phase the delivery

### Phase 0 — Evidence and access

Deliver:

- agreed outcome and baseline;
- current-state workflow;
- source-of-truth/integration map;
- risk and assumption register;
- security/data classification;
- sample dataset and evaluation plan;
- pilot cohort and owner;
- success, stop, and rollback criteria.

Exit only when access and authority are real, not promised.

### Phase 1 — Offline proof

Use historical, redacted, or approved data. Build the smallest pipeline that tests feasibility.

Examples:

- parse 200 representative invoices;
- retrieve evidence for a fixed question set;
- map ERP cost codes;
- simulate a discharge follow-up queue;
- measure quality by risk slice.

No production side effects. Compare against a deterministic or current-process baseline.

### Phase 2 — Shadow mode

Run on live inputs without influencing production decisions. Capture:

- proposed outputs;
- latency and cost;
- missing data;
- disagreement with actual human outcomes;
- integration errors;
- slice-level quality;
- would-have-triggered actions.

Shadow mode tests reality but not adoption. A user may behave differently when responsible for the result.

### Phase 3 — Assisted pilot

A bounded cohort sees cited recommendations. Humans retain authority and execute actions manually or through approval-bound tools.

Limit:

- named users and projects/facilities;
- supported case types;
- operating hours;
- data scope;
- model and workflow versions;
- maximum action consequence;
- escalation owner.

Collect structured feedback and downstream outcomes, not only thumbs-up signals.

### Phase 4 — Controlled automation

Automate reversible, well-evaluated steps first. Consequential external writes require current authorization, immutable approval-bound plans, idempotency, outcome-unknown handling, and reconciliation.

Use canary cohorts, budgets, kill switches, and rollback. Preserve a manual path.

### Phase 5 — Scale and productize

Only after pilot evidence:

- harden multi-tenancy and self-service configuration;
- standardize connectors and data contracts;
- define SLOs and on-call;
- automate tenant onboarding and evaluation;
- document runbooks and support boundaries;
- retire pilot shortcuts;
- extract reusable primitives;
- define commercial and capacity model.

Do not confuse “works for one expert-assisted tenant” with a scalable product.

## Architecture decision record

For major choices capture:

- context and problem;
- decision;
- alternatives;
- assumptions;
- consequences;
- security/reliability impact;
- evidence;
- owner and review date.

Examples include polling versus webhook, read-only versus write automation, managed model versus self-hosted model, tenant-isolation strategy, and manual approval versus automatic action.

An ADR is useful only if it records why the decision was reasonable at that time and what evidence would change it.

## Pilot contract

A pilot needs an explicit working agreement:

- users, sites, tenants, and workflows included;
- data and system access;
- supported hours and escalation;
- responsibility for decisions and approvals;
- known limitations;
- success metrics and baseline;
- evaluation method and sample size;
- prohibited actions;
- incident and rollback path;
- start, review, and exit dates;
- data retention after the pilot;
- who pays ongoing integration/operations cost.

Without scope and exit criteria, a pilot can become unsupported production.

## Success metrics and guardrails

Use a metric tree.

**Business outcome**

- cycle time, deadline attainment, rework, cost, collections, safety, or access.

**Adoption**

- eligible cases shown, reviewed, accepted, completed, and returned to workaround.

**Quality**

- groundedness, extraction accuracy, correction rate, false accept/reject, outcome quality by slice.

**Reliability**

- freshness, availability, latency, backlog, reconciliation age, incident rate.

**Risk guardrails**

- unauthorized access, tenant leak, duplicate action, stale approval, unsupported clinical action.

A pilot succeeds only if business value and guardrails both meet thresholds. High model accuracy with zero operator adoption is not success.

Avoid vanity metrics such as messages sent, documents indexed, or users invited unless they connect to outcomes.

## Design for partial failure

Trace each boundary:

```text
request → identity/policy → source read → AI/logic → review
        → durable command → external write → confirmation/reconciliation
```

For every step ask:

- what can time out?
- can it be retried?
- is the outcome known?
- what is the idempotency key?
- what state is durable?
- what does the user see?
- who is alerted?
- how is truth reconciled?

A FDE should be able to narrate the system during failure, not only during the demo.

## Data and AI evaluation plan

Build the evaluation set from the deployment distribution:

- common cases;
- costly exceptions;
- low-quality scans;
- new vendors or locations;
- multilingual content;
- missing and contradictory evidence;
- adversarial or unsafe instructions;
- permission boundaries;
- changed documents and stale approvals.

Separate development, validation, and post-launch monitoring sets. Preserve source and label lineage.

Evaluate the entire workflow:

- ingestion and parsing;
- retrieval and authorization;
- answer/proposal quality;
- tool argument validity;
- policy and approval behavior;
- execution/reconciliation;
- human correction and business outcome.

Do not let a fluent answer hide missing source evidence.

## Human workflow and adoption

A new system changes roles, incentives, and accountability.

Ask:

- who gains work and who loses work?
- does the system interrupt the operator’s existing queue?
- is evidence visible at the decision point?
- can the user correct the result?
- does feedback lead to a visible improvement?
- who owns exceptions?
- what training and office hours are needed?
- what happens when users bypass the system?

Embed into existing work where possible. A separate dashboard that requires users to remember to visit may not change outcomes.

Automation bias and distrust can coexist. Show uncertainty and provenance; do not force blind acceptance or make correction punitive.

## Security and regulated deployment

Before production, establish:

- tenant and resource authorization;
- workload identity and least-privilege scopes;
- data classification and residency;
- encryption and secret rotation;
- approved model/provider use;
- retention and deletion;
- audit and support access;
- threat model, including prompt injection;
- incident ownership;
- vendor agreements and subprocessors;
- healthcare role and minimum-necessary boundaries where applicable.

A pilot is not exempt from security because it is small. Reduce risk through narrower scope, isolated environments, synthetic data, and read-only access.

## Handoff and operational ownership

A production handoff includes:

- architecture and data-flow diagrams;
- service and data owners;
- SLOs and dashboards;
- runbooks and escalation;
- deployment and rollback;
- configuration and secret rotation;
- reconciliation and backfill procedures;
- known limitations;
- cost and capacity model;
- support/training material;
- outstanding risks and roadmap.

Use paired operation before exit: customer or product team handles incidents and routine changes while the FDE observes.

A system that only the FDE can operate is unfinished.

## Construction example

A contractor asks for “an AI agent that automates invoice approval.”

Discovery finds:

- invoices arrive by email and vendor portal;
- purchase orders are in the ERP;
- change orders are current in the project platform;
- duplicate checks are manual;
- cost-code exceptions are resolved through email;
- project managers approve coding, controllers approve payment;
- ERP writes can time out without a clear result;
- month-end volume is four times normal.

Outcome statement:

> Reduce median invoice-ready-for-approval time while preserving duplicate prevention, separation of duties, and accurate ERP posting.

Phased delivery:

1. Baseline a stratified sample by project, vendor, amount, and scan quality.
2. Build an offline evidence packet: extracted fields, source citations, PO balance, change-order status, and duplicate candidates.
3. Shadow live invoices and compare proposals with completed human decisions.
4. Pilot with two projects in read-only assisted mode.
5. Add structured corrections and queue metrics.
6. Enable approval-bound ERP coding for low-risk invoices with stable operation IDs and reconciliation.
7. Keep high-value, missing-evidence, and new-vendor cases in human review.
8. Reconcile daily to ERP totals and study correction/outcome rates.
9. Productize mappings, tenant configuration, evaluation, onboarding, and support only after evidence meets gates.

The first deliverable is not autonomous payment. It is trusted, cited evidence at the point of approval.

## Healthcare operations variation

A hospital asks for an AI assistant to improve discharge follow-up.

Discovery separates:

- clinical decisions;
- patient education;
- appointment scheduling;
- transportation and social-service coordination;
- operational queue management.

The FDE maps the authoritative clinical record, scheduling system, care-team roles, identity boundaries, update latency, correction behavior, and existing escalation. A scheduler cannot become the approver of a clinical recommendation.

The first phase may create a read-only operational queue from approved fields, with source citations and current encounter versions. Shadow mode compares proposed routing with actual follow-up. An assisted pilot starts with one service line and operating window.

Clinical recommendations remain under licensed review. PHI is minimized in model calls and telemetry. Late corrections update the queue. If authoritative data or policy is unavailable, the system degrades to a clearly labeled manual workflow rather than inventing a clinical answer.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Long discovery | More context | Delayed evidence and momentum |
| Rapid prototype | Fast learning | Can bypass real constraints |
| Read-only first | Lower risk, validates data | Less immediate automation value |
| Write automation first | Visible ROI | High integration and outcome risk |
| Thin vertical slice | Tests end-to-end reality | Narrow apparent scope |
| Broad mock demo | Stakeholder excitement | Defers hard boundaries |
| One-tenant customization | Fast customer fit | Product fragmentation |
| Standard capability | Reuse and supportability | May miss critical local workflow |
| Shadow mode | Live technical evidence | Does not prove adoption |
| Human-assisted pilot | Safer accountability | Labor and queue latency |
| Custom model | Control and specialization | Data, evaluation, serving cost |
| Managed model | Fast capability | Provider, privacy, and cost dependency |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Start from requested feature | Solve the wrong problem | Outcome and workflow framing |
| Interview only executives | Miss operational exceptions | Observe frontline work |
| No baseline | ROI cannot be proven | Instrument or sample before pilot |
| Average hides hard slices | Production quality collapse | Stratified evaluation |
| “API exists” assumed sufficient | Integration blocked late | Early read/write spike |
| No source-of-truth map | Conflicting records | Entity authority and freshness |
| Prototype uses clean handpicked data | Demo does not generalize | Representative evaluation set |
| Shadow success treated as adoption | Workflow ignored at launch | Assisted pilot with operators |
| Pilot has no boundary | Unsupported production | Written scope and exit |
| Model metric is sole success metric | No business improvement | Outcome and adoption tree |
| External timeout treated as failure | Duplicate action | Outcome-unknown reconciliation |
| Customer-specific fork | Upgrade debt | Reusable typed primitives |
| Security postponed until scale | Sensitive-data incident | Narrow secure pilot from start |
| No manual/degraded path | Outage stops operations | Explicit fallback and ownership |
| FDE owns every alert forever | Handoff failure | Paired operation and runbooks |
| Rollback only covers software | External effects persist | Business reconciliation plan |

## What to measure

Track assumption closure, access lead time, sample representativeness, baseline coverage, pilot enrollment and eligible volume, adoption funnel, cycle-time distribution, rework and exception rate, quality by slice, evidence/citation use, correction reasons, queue age, integration error, outcome-unknown operations, reconciliation age, reliability SLOs, support burden, cost per completed outcome, security guardrails, configuration forks, time to onboard the next tenant, and unresolved ownership.

Maintain a decision log connecting each scope or architecture change to new evidence.

## Interview prompt

> A mid-market contractor asks you, as an FDE, to deploy an AI agent that automates invoice approvals across email, a project-management platform, and an ERP. Requirements are ambiguous, source data conflicts, month-end volume spikes, and ERP writes can time out. Explain discovery, architecture, pilot phases, success metrics, production hardening, and handoff. Then adapt the plan to a hospital discharge-follow-up workflow.

Cover stakeholder ownership, current-state mapping, baseline, assumptions, source of truth, integration spikes, read-only/shadow/assisted/automated phases, evaluation, security, reliability, adoption, rollback, reconciliation, productization, and support.

## Worked answer

**One-line conclusion:** An FDE should convert ambiguity into a measured, reversible delivery path: prove the workflow and data first, automate consequential actions only after evidence, and leave behind an operable product—not a permanent pilot.

“I would begin with the operational outcome and owner, then shadow project managers and controllers to map triggers, handoffs, sources, exceptions, and irreversible actions. I would baseline eligible volume, cycle time, rework, duplicate risk, and month-end slices. The source map would name the ERP for invoice/payment truth, the project platform for current change orders, identity systems for roles, and the workflow service for approvals.

I would attack access and idempotency assumptions with read-only API and sandbox write spikes. Phase 1 builds an offline cited evidence packet on representative historical invoices. Phase 2 shadows live cases. Phase 3 pilots read-only recommendations for two projects with defined users, hours, success thresholds, security scope, and stop criteria. Only after adoption and slice-level quality pass would I enable approval-bound writes for low-risk cases. Execution rechecks roles and source versions, uses stable operation IDs, and reconciles ERP timeouts before retry.

Metrics include median and tail cycle time, eligible-to-completed adoption, correction and exception rates, duplicate prevention, queue age, model/retrieval quality by scan and vendor slice, integration reliability, reconciliation age, and cost per completed invoice. A kill switch preserves read-only evidence. The handoff includes SLOs, dashboards, runbooks, mappings, secret rotation, backfills, support owners, and known limits. Healthcare follows the same phased method but separates operational scheduling from clinical authority, minimizes PHI, and retains licensed review.”

## Practice lab

A contractor wants full invoice automation in six weeks:

- invoices arrive through email and a vendor portal;
- the ERP has purchase orders, payments, and the final ledger;
- change orders are fresher in a project platform;
- cost-code exceptions live in spreadsheets and email;
- scanned document quality varies;
- month-end volume is 4× normal;
- project managers and controllers have separate approval roles;
- the ERP supports writes but its timeout semantics are unclear;
- leaders claim each invoice takes 20 minutes, but no baseline exists.

Create:

- an outcome statement and non-goals;
- stakeholder and source-of-truth maps;
- the first ten discovery questions;
- baseline and stratified sampling plan;
- top assumptions and cheapest tests;
- a Phase 0–5 delivery plan;
- pilot contract and success/stop gates;
- architecture boundaries and failure handling;
- rollout, rollback, and reconciliation plan;
- handoff package;
- healthcare adaptation for discharge follow-up.

## References

- [Google Cloud Architecture Framework — System design](https://cloud.google.com/architecture/framework/system-design)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [UK Government Service Manual — Discovery phase](https://www.gov.uk/service-manual/agile-delivery/how-the-discovery-phase-works)
- [Thoughtworks Technology Radar — Fitness functions](https://www.thoughtworks.com/radar/techniques/architectural-fitness-function)
