# Online experiments, shadow deployments, and causal measurement

Offline evaluation tells you whether a candidate can work on a controlled dataset. A production experiment asks a different question:

> Did exposing eligible users or workflows to this change cause a meaningful improvement without unacceptable harm?

Answering that requires experimental design, trustworthy telemetry, operational safeguards, and patience for delayed outcomes.

## Mental model

Treat an online experiment as a **versioned causal contract**, not a feature flag with a dashboard.

The contract defines:

- hypothesis;
- eligible population;
- experimental unit;
- randomization and assignment;
- treatment and control bundles;
- exposure event;
- primary outcome;
- guardrail metrics;
- minimum detectable effect;
- analysis horizon;
- stopping and rollback rules;
- interference and exclusion assumptions;
- data-quality checks;
- accountable owner and approvers.

A useful causal picture is:

[
ATE = E[Y(1) - Y(0)]
]

where (Y(1)) is the outcome an experimental unit would have under treatment and (Y(0)) under control. You can observe only one potential outcome for each unit, so randomization creates comparable groups in expectation.

The central invariant is:

> Assignment, exposure, treatment identity, and outcome must be joined for the same eligible unit without post-treatment leakage.

## When to use it

Use online experiments when:

- a product or workflow change is safe enough for controlled exposure;
- business impact cannot be established from offline metrics alone;
- user or operator behavior may change;
- the model affects ranking, recommendations, review routing, or assistance;
- latency, cost, workload, and downstream outcomes matter;
- a release has passed offline and shadow gates;
- enough eligible units and observation time exist.

Do not randomize a change that lacks basic safety evidence or can cause irreversible harm without a strong containment and approval plan.

Do not use an A/B test when:

- legal, ethical, contractual, or clinical requirements prohibit withholding treatment;
- traffic is too small for an informative result;
- spillovers make individual randomization invalid;
- the change is infrastructure-only and can be validated directly;
- a severe bug must be fixed for everyone;
- the decision is better answered by a staged rollout, simulation, or qualitative research.

## Shadow, canary, A/B, switchback, and interleaving

These answer different questions.

| Method | User-visible? | Measures | Main limitation |
| --- | --- | --- | --- |
| Offline replay | No | Historical task quality | Dataset and policy mismatch |
| Shadow deployment | No | Candidate output, latency, compatibility | No behavioral or workflow effect |
| Canary rollout | Yes, small cohort | Operational safety | Usually not powered for causality |
| A/B test | Yes | Causal outcome difference | Needs valid randomization and sample |
| Switchback | Yes, by time block | Marketplace/system-wide treatment | Time trends and carryover |
| Cluster randomization | Yes, by group | Effects with within-group spillover | Fewer independent units |
| Interleaving | Yes, mixed ranking | Sensitive preference comparison | Best for rankers, not all outcomes |

A shadow is not a control group. It sees production inputs but does not create the alternative user behavior or downstream workflow that treatment would cause.

A canary is not automatically an experiment. If assignment follows geography, employee preference, or risk, treatment and control may differ before exposure.

## Hypothesis and estimand

Write the hypothesis in operational terms.

Weak:

> The new model is better.

Stronger:

> For eligible subcontractor invoices submitted during the experiment, treatment reduces median reviewer handling time by at least 10% over 14 days without increasing incorrect auto-routing by more than 0.2 percentage points, duplicate ERP proposals, or p95 latency beyond the approved threshold.

Define the estimand:

- intention-to-treat: effect of assignment;
- treatment-on-treated: effect among actually exposed units;
- per-protocol: effect among units following the protocol;
- average effect for a named population;
- effect by predeclared segments.

Intention-to-treat usually preserves randomization and is the primary product estimate. Exposure analysis can diagnose delivery, but conditioning on post-assignment behavior can introduce bias.

## Experimental unit

Choose the smallest independent unit that matches how treatment works.

Possible units:

- user;
- organization or tenant;
- project;
- vendor;
- document;
- patient;
- clinic;
- time block.

Randomizing documents is invalid if reviewers learn from treatment and apply it to control documents. Randomizing users may contaminate results if a project team collaborates in one shared queue. Then project- or tenant-level assignment may be safer.

Cluster randomization reduces contamination but lowers statistical power because units within a cluster are correlated. Analyze at the assignment level or use methods that account for clustering.

Document why one unit’s treatment cannot materially change another unit’s outcome. This is the interference assumption; ignoring it creates confident but wrong estimates.

## Eligibility and assignment

Eligibility is evaluated before treatment using pre-treatment facts:

- tenant and contract;
- region and policy;
- workflow state;
- document/task type;
- model capability;
- privacy and risk class;
- required integration availability.

Use stable deterministic assignment such as a keyed hash:

[
bucket = H(experiment_id, unit_id, secret_salt) mod 10000
]

Map fixed ranges to variants. A secret or protected salt prevents clients from choosing their bucket.

Requirements:

- assignment is stable across retries and devices;
- tenant identity is trusted;
- allocations are versioned;
- mutually exclusive experiments coordinate namespaces;
- stratification variables are pre-treatment;
- reassignment is rare and recorded;
- bots, employees, or test traffic follow declared rules.

Do not hash on an event ID if the same user or project must receive consistent treatment.

## Exposure logging

Assignment does not mean exposure.

Log exposure when the treatment could first affect the unit—for example, when the reviewer actually sees the new ranking—not when the feature service checks a flag.

An exposure event includes:

- experiment and variant;
- unit and tenant;
- assignment version;
- timestamp;
- released treatment bundle;
- eligibility decision;
- surface/workflow;
- request or operation correlation ID.

Make exposure idempotent. Duplicate client events must not inflate sample size.

Server-side exposure is usually more reliable for backend behavior. Client-side exposure may be required for rendered UI, but it needs offline delivery, deduplication, and bot handling.

Run sample-ratio mismatch checks: if a 50/50 experiment produces 54/46 without explanation, stop analysis and investigate assignment, logging, eligibility, or loss.

## Treatment definition

An AI treatment is a bundle, not only a model name. Pin:

- model and runtime;
- prompt and context compiler;
- retrieval and index generation;
- policy and tool schemas;
- feature versions;
- UI and workflow behavior;
- review thresholds;
- fallback;
- caching;
- deployment region.

If any component changes mid-experiment, create a new treatment version or predeclare the change. Otherwise the estimated effect averages unknown configurations.

Log the actual resolved bundle per exposure.

## Metrics hierarchy

Use a small predeclared hierarchy.

### Primary outcome

The decision metric tied to user or business value, such as:

- reviewer minutes per correctly resolved invoice;
- percentage of RFIs resolved within target time;
- claim packet completion without rework;
- appointment attendance after outreach.

### Guardrails

Metrics that must not worsen beyond limits:

- incorrect or unsafe action;
- authorization or privacy incident;
- duplicate external effect;
- critical-slice quality;
- p95/p99 latency;
- timeout and error rate;
- reviewer backlog;
- user complaint or override;
- cost per successful outcome;
- fairness or access disparity.

### Diagnostic metrics

Explain mechanism:

- retrieval coverage;
- citation use;
- suggestion acceptance;
- edit distance;
- model fallback;
- tool proposal rate;
- cache hit;
- review time components.

Do not promote from a diagnostic metric while the primary outcome is neutral or harmful unless the decision rule explicitly permits it.

## AI-specific outcome measurement

AI outcomes are often delayed and partially observed.

Examples:

- an invoice routed today becomes disputed weeks later;
- an RFI answer is corrected after field work;
- a denied claim is overturned later;
- a reviewer accepts a suggestion but edits it subtly.

Separate:

- immediate model output;
- operator interaction;
- downstream workflow;
- mature business outcome.

Predefine label maturity and attribution windows. Do not count unresolved recent cases as successes.

For qualitative outputs, use:

- deterministic checks;
- blinded expert review;
- outcome-linked corrections;
- carefully validated model judges;
- stratified sampling;
- adjudication of disagreements.

LLM-as-judge can be a diagnostic measure, not the sole basis for a consequential launch. Blind judges to variant and randomize presentation order.

## Power, duration, and minimum detectable effect

Before launch, estimate:

- baseline rate or variance;
- meaningful effect size;
- significance level;
- desired power;
- allocation;
- cluster correlation if applicable;
- attrition and delayed labels;
- expected eligible traffic.

A huge sample can make trivial effects statistically significant; a small sample can miss valuable effects. Decide practical significance first.

Run long enough to cover operational cycles such as weekdays, payroll, month-end, or clinic schedules. Avoid stopping immediately after the first significant result.

For ratio metrics, heavy-tailed costs, or clustered workflows, validate the analysis with simulation or robust methods rather than applying a simple formula blindly.

## Sequential testing and peeking

Repeatedly checking a fixed-horizon p-value and stopping when it crosses 0.05 inflates false positives.

Choose one:

- fixed-horizon analysis with a locked end;
- preplanned group-sequential boundaries;
- always-valid confidence sequences or sequential methods;
- Bayesian decision rules with explicit priors and loss.

Operational safety monitoring is continuous and separate from efficacy peeking. Stop immediately for a predeclared severe guardrail breach regardless of the efficacy plan.

Record every allocation, metric, and stopping-rule change.

## Variance reduction

Pre-treatment covariates can improve precision without changing treatment.

Common methods:

- stratified randomization;
- regression adjustment;
- CUPED using a correlated pre-experiment metric;
- blocking by tenant size, workflow, or baseline rate.

Covariates must be measured before treatment and specified before analysis. Do not choose them after seeing which version wins.

Variance reduction cannot repair broken assignment, interference, or missing outcomes.

## Multiple metrics and segments

Testing many metrics, variants, time windows, and segments creates false discoveries.

Controls:

- one primary outcome;
- limited predeclared secondary metrics;
- family-wise or false-discovery adjustment where appropriate;
- predeclared segment hypotheses;
- confidence intervals, not winner labels alone;
- separate exploratory analysis marked as exploratory;
- replication for surprising subgroup effects.

Segments should be large enough and meaningful. A positive average can hide harm to handwriting, Spanish-language, small-contractor, or rural-clinic slices. Guardrail slices may block launch even when not powered as primary hypotheses.

## Network effects, carryover, and switchbacks

Some systems have shared capacity or marketplace effects:

- treatment increases reviewer queue load for control;
- a smarter dispatcher changes equipment availability;
- a model uses a shared cache or GPU pool;
- clinic outreach changes call-center congestion.

Individual randomization may violate independence.

Options:

- randomize by project, tenant, clinic, or region;
- reserve isolated capacity;
- use switchback time blocks;
- model spillovers explicitly;
- analyze saturation at different treatment proportions.

Switchbacks require randomized time blocks, washout between treatments when needed, and adjustment for hour/day trends. They are weak when effects persist long after the block.

## Shadow deployments

A shadow system duplicates eligible read traffic to a candidate without affecting users.

Use it to test:

- input and schema compatibility;
- latency, throughput, and cost;
- output agreement and disagreement;
- structured validity;
- retrieval and citation behavior;
- fallback and timeout;
- resource saturation;
- unexpected tool proposals.

Rules:

- never duplicate external side effects;
- replace mutating tools with simulators or read-only validation;
- maintain authorization and data-residency rules;
- tag shadow traffic so it does not pollute business analytics;
- bound extra load and disable shadowing during pressure;
- redact/minimize logs;
- sample representative traffic;
- compare only when control and candidate receive equivalent valid inputs.

Shadow results can reveal candidate defects, but agreement with the incumbent is not truth. The incumbent may also be wrong.

## Experiment safety for tools and agents

For AI systems that propose or execute actions, separate experiments:

1. **reasoning shadow** — compare plans without executing;
2. **proposal experiment** — users see suggestions, but humans execute;
3. **approval experiment** — treatment changes review/routing;
4. **bounded action experiment** — only low-risk, reversible, idempotent tools;
5. **broader action rollout** — after causal and safety evidence.

Every treatment must still pass authorization, current policy, approval binding, schema validation, and idempotency. Experiment assignment never grants tool permission.

Do not send the same shadow write to an ERP. Use a simulator, dry-run endpoint, or compare proposed canonical operations.

An outcome-unknown tool call is reconciled normally; never retry merely to preserve the experiment.

## Experiment platform

A reliable platform provides:

- registry for hypotheses, owners, units, metrics, and dates;
- namespace and collision management;
- eligibility and assignment service;
- versioned configuration;
- exposure and outcome event schemas;
- metric computation with late-data handling;
- sample-ratio and instrumentation checks;
- guardrail alerting and kill switch;
- analysis notebooks or service with reproducible code;
- privacy, retention, and access controls;
- decision record and archive.

Separate the real-time assignment path from analytical computation. Assignment must be highly available and fail safely. If unavailable, use a declared default—usually control or no exposure—and record it.

The warehouse recomputes metrics from immutable events. Dashboards are views, not the only record of the experiment.

## Data quality

Before interpreting effects, verify:

- assignment balance;
- sample-ratio match;
- exposure coverage;
- duplicate and missing events;
- variant identity;
- event ordering and clock skew;
- outcome maturity;
- tenant/user joins;
- bot/internal traffic;
- metric definition version;
- pipeline delays;
- exclusion reasons.

Run an A/A test periodically: both groups receive the same treatment. It tests randomization, logging, variance, false-positive behavior, and analysis—not whether future treatment effects are correct.

## Construction operations example

A construction platform tests a new AI invoice-review workspace.

Control:

- existing extraction;
- chronological review queue;
- evidence opened manually.

Treatment:

- upgraded extraction;
- risk-ranked queue;
- cited change-order and contract evidence;
- suggested discrepancy reasons;
- no automatic ERP write.

Design:

1. randomize by project, not invoice, because reviewers collaborate within project queues;
2. stratify by tenant and project invoice volume;
3. expose when a reviewer first opens the treatment workspace;
4. primary outcome: reviewer minutes per correctly resolved invoice over mature cases;
5. guardrails: missed high-risk discrepancy, duplicate proposal, incorrect project/vendor, p95 latency, reviewer backlog, and cost;
6. diagnostics: citation opening, suggestion acceptance/edit, extraction disagreement, and fallback;
7. shadow the candidate first with ERP tools simulated;
8. run through at least one month-end cycle;
9. use intention-to-treat as primary analysis;
10. account for clustering by project;
11. stop immediately for authorization leakage or a duplicate external effect;
12. do not expose auto-posting until a separate approval-bound action experiment passes.

If treatment sends difficult invoices to senior reviewers while control does not, raw completion time may reflect staffing changes. Reviewer assignment must be part of the treatment definition and causal interpretation.

## Healthcare operations variation

A health system tests an AI-assisted appointment outreach workflow.

Randomize by clinic or scheduling team if staff behavior would contaminate patient-level assignment. The treatment suggests outreach channel and timing; it does not reduce access or cancel appointments.

Primary outcome might be completed appointments among eligible scheduled visits after the label window. Guardrails include opt-out, complaint, failed contact, scheduling error, workload, language equity, and differences across protected or operationally vulnerable groups.

PHI stays within approved services. Exposure and analytics data are minimized and purpose-limited. Clinical or coverage decisions are outside the experiment unless governed under the appropriate clinical, legal, ethics, and research processes.

A statistically significant attendance increase is insufficient if it increases burden, excludes a language group, or shifts work unsafely to staff.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Shadow test | Safe production inputs | No behavioral effect |
| Canary | Fast safety signal | Confounded causal estimate |
| User randomization | High statistical power | Cross-user contamination |
| Cluster randomization | Reduces spillover | Larger sample needed |
| Switchback | Handles shared systems | Time/carryover complexity |
| Fixed horizon | Simple inference | Slower decisions |
| Sequential design | Earlier valid decisions | More design/analysis complexity |
| One primary metric | Clear decision | Misses some nuance |
| Many metrics | Broad visibility | False discovery |
| Short proxy metric | Fast feedback | May not predict business outcome |
| Mature outcome | High validity | Slow experiment |
| 50/50 allocation | Maximum power | More treatment exposure |
| Small treatment share | Lower risk | Longer experiment |
| CUPED adjustment | Better precision | Needs valid pre-treatment covariate |
| Tenant-level assignment | Isolation | Few independent units |
| Automated analysis | Consistency | Can hide instrumentation failures |
| Human decision review | Context and accountability | Delay and subjectivity |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Treat canary as randomized | Confounded effect | Explicit assignment design |
| Log assignment as exposure | Diluted/misleading estimate | Log actual opportunity to affect |
| Randomize invoice within shared queue | Spillover | Project/tenant clustering |
| Change treatment mid-test | Uninterpretable average | Pin bundle/version |
| Count recent unresolved cases as success | Immature-label bias | Attribution and maturity window |
| Stop on first p<0.05 | False positive | Predeclared sequential/fixed rule |
| Inspect dozens of segments | Spurious winners | Multiplicity control |
| Optimize proxy only | Business harm | Primary outcome and guardrails |
| Condition primary analysis on acceptance | Selection bias | Intention-to-treat |
| Duplicate exposure events | Inflated sample | Idempotent event schema |
| Missing treatment logging | Unknown delivered variant | Runtime bundle identity |
| Sample-ratio mismatch ignored | Broken assignment/logging | Automatic SRM stop |
| Shared GPU/reviewer capacity | Control contaminated | Isolate or model interference |
| Shadow executes tool | Duplicate external effect | Simulator/read-only tools |
| Treatment grants permission | Security breach | Independent policy/authorization |
| A/A test fails but launch continues | Invalid platform | Repair instrumentation first |
| Average gain hides slice harm | Unsafe inequity | Predeclared guardrail slices |
| Experiment ends before month-end | Nonrepresentative result | Cover business cycles |
| Rollout follows significance alone | Operational regression | Practical effect plus safety review |

## What to measure

Track eligible, assigned, exposed, and analyzed units; allocation and sample-ratio mismatch; exposure loss and duplicates; treatment bundle identity; primary effect and confidence interval; guardrail breaches; practical versus statistical significance; segment effects; label maturity; attrition; interference indicators; latency and cost; review/approval workload; tool proposals and external effects; experiment duration; stop reasons; percentage of experiments with reproducible decision records; and post-launch effect persistence.

The goal is not to maximize the number of experiments. It is to make better decisions with known uncertainty and bounded risk.

## Interview prompt

> Design an experimentation platform for a multi-tenant construction AI system. Teams want to compare extraction models, evidence retrieval, reviewer ranking, and agent tool proposals. Outcomes are delayed, reviewers collaborate within projects, and ERP actions can be irreversible. Adapt the design for healthcare appointment outreach with PHI.

Cover shadow traffic, canaries, randomization units, assignment, exposure logging, metrics, power, sequential testing, interference, delayed labels, AI evaluation, tool safety, multi-tenancy, data quality, and decision records.

## Worked answer

**One-line conclusion:** Randomize the correct independent unit, log real exposure to an immutable treatment bundle, measure predeclared mature outcomes and guardrails, and keep shadow or experimental AI actions behind normal authorization and side-effect controls.

“I would begin with an experiment registry that fixes the hypothesis, eligible population, assignment unit, treatment bundle, primary outcome, guardrails, MDE, duration, analysis method, and stopping rules. For construction invoice review I would randomize projects or project teams, because invoice-level assignment would contaminate shared reviewer queues.

A keyed hash produces stable tenant-scoped assignment. Exposure is logged when the reviewer first sees the treatment, with experiment, unit, assignment version, and exact model/prompt/retrieval/policy/UI bundle. Assignment remains the primary intention-to-treat analysis; exposure diagnostics verify delivery.

Before user exposure, a bounded shadow compares schema validity, citations, latency, cost, and proposed tool calls. Mutations use simulators—no ERP write is duplicated. Canary rollout then tests safety, but only a randomized phase estimates causal business impact.

The primary metric is reviewer time per correctly resolved mature invoice. Guardrails include missed discrepancies, authorization, duplicate proposals, latency, backlog, cost, and critical document slices. I would calculate sample size from baseline variance and a practical MDE, cover month-end, account for project clustering, use a predeclared fixed or sequential rule, and investigate any sample-ratio mismatch before interpreting results.

Outcome events are immutable and recomputed in the warehouse with versioned metric code. I would monitor eligibility, assignment, exposure, label maturity, interference, variant identity, and guardrails continuously. Promotion requires practical improvement, trustworthy instrumentation, no safety breach, and accountable review—not statistical significance alone.

For healthcare I would choose clinic-level assignment when staff behavior spills over, minimize PHI in experiment telemetry, measure attendance and workload, enforce language/access guardrails, and never let the experiment restrict care.”

## Practice lab

Design an experiment for a new invoice-review assistant when:

- reviewers share project queues;
- the candidate changes extraction, ranking, citations, and UI;
- month-end volume differs from normal weeks;
- only 70% of assigned users open the new workspace;
- reviewer time is heavy-tailed;
- disputes mature 30 days later;
- the treatment increases suggestion acceptance but may miss handwritten discrepancies;
- both variants share a saturated GPU pool;
- a shadow candidate proposes an ERP write;
- a dashboard shows significance after repeated daily peeking;
- one small tenant appears to improve dramatically.

Specify the unit, assignment, exposure event, treatment bundle, hypothesis, primary metric, guardrails, MDE/power inputs, analysis horizon, clustering, sequential rule, shadow-tool policy, interference mitigation, label maturity, segment policy, and launch decision.

Then adapt it to appointment outreach across clinics with PHI, staff spillovers, multilingual patients, opt-outs, and a rule that no experiment may reduce access to care.

## References

- [Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing](https://experimentguide.com/)
- [Microsoft ExP Platform: An experimentation platform for large-scale online controlled experiments](https://www.microsoft.com/en-us/research/publication/online-experimentation-at-microsoft/)
- [CUPED: Improving the Sensitivity of Online Controlled Experiments](https://www.microsoft.com/en-us/research/publication/improving-the-sensitivity-of-online-controlled-experiments-by-utilizing-pre-experiment-data/)
- [Google: Overlapping Experiment Infrastructure](https://research.google/pubs/overlapping-experiment-infrastructure-more-better-faster-experimentation/)
- [A Survey of Network Experimentation](https://arxiv.org/abs/1904.10538)
- [NIST AI 600-1: Generative Artificial Intelligence Profile](https://doi.org/10.6028/NIST.AI.600-1)
