# Model cascades, ensembles, routing, and abstention under uncertainty

A production AI system rarely needs the largest model for every request. It also should not force an answer when evidence, capability, authorization, or confidence is inadequate.

Cascades, ensembles, routers, and abstention policies allocate work across models and humans while preserving quality, safety, latency, privacy, and cost.

## Mental model

Treat the system as a **risk-aware decision policy over capabilities**, not a model picker.

For each request, the policy must decide:

1. is this request eligible for automated processing?
2. which models, tools, regions, and data paths are allowed?
3. what is the cheapest sufficient path?
4. how will sufficiency be measured?
5. when should the system escalate, ask for more evidence, or abstain?
6. which final output and lineage will be returned?
7. how will failures and uncertain outcomes be reconciled?

Use two stages:

### Hard eligibility

Filter by non-negotiable constraints:

- authorization and tenant scope;
- task and modality capability;
- data residency and privacy;
- model/provider approval;
- context and output limits;
- tool permissions;
- risk tier;
- contractual restrictions;
- current health and revocation status.

### Optimization

Among eligible paths, optimize:

- evaluated quality;
- latency/deadline;
- cost;
- capacity;
- energy or resource use;
- user preference;
- fairness and tenant budgets.

The core invariant is:

> A cheaper or faster model is never eligible merely because it is available; it must first satisfy the task’s policy, capability, and evidence requirements.

## When to use it

Use cascades or routing when:

- requests vary substantially in difficulty;
- multiple models have different quality/cost/latency profiles;
- simple cases can be handled deterministically or by a smaller model;
- long contexts, images, or languages require specialized models;
- privacy or region limits eligible providers;
- capacity must be shared fairly;
- human review is scarce;
- abstention is safer than guessing;
- workloads include both interactive and asynchronous paths.

Use ensembles when independent or complementary models provide meaningful error reduction and the extra cost/latency is justified.

Do not add a router when one model already meets requirements economically and reliably. Routing adds another model, policy, dataset, failure mode, and evaluation surface.

## Architecture

A typical request path:

```text
authenticated request
        |
policy + capability filter
        |
difficulty/risk estimation
        |
route or cascade stage
        |
validation + evidence checks
        |
answer / escalate / human review / abstain
        |
decision and outcome log
```

Keep the router outside the generative model’s authority. A model may provide diagnostic signals, but trusted code enforces eligibility, budgets, deadlines, and tool permissions.

The router and all candidate paths are part of one versioned release bundle.

## Cascades

A cascade runs stages sequentially, stopping when one meets an acceptance contract.

Example:

1. deterministic parser;
2. small specialized extraction model;
3. larger multimodal model;
4. expert review.

A stage should return:

- candidate output;
- structured validation;
- confidence/uncertainty features;
- evidence and citations;
- reason codes;
- resource usage;
- accept/escalate decision inputs.

The cascade policy—not the stage itself—decides whether to stop.

### Advantages

- cheap/easy cases exit early;
- scarce models and humans serve difficult cases;
- latency and cost can improve;
- each stage can specialize.

### Risks

- early-stage confident errors never escalate;
- later stages inherit corrupted intermediate outputs;
- tail latency accumulates;
- queueing at the expensive stage creates nonlinear delay;
- per-stage metrics can hide end-to-end failure;
- a stage may use data unavailable to another path.

Measure the **oracle gap**: cases where the system stopped early but a later eligible stage would have corrected it.

## Parallel ensembles

An ensemble runs multiple models and combines outputs.

Methods:

- majority vote;
- probability averaging;
- weighted voting;
- stacking or meta-model;
- rank aggregation;
- agreement/disagreement rule;
- evidence union with conflict detection;
- judge model over candidates.

Ensembles help only when errors are sufficiently diverse. Five models trained on the same data and architecture may agree on the same mistake.

For structured extraction, combine at field level rather than whole document. For generated answers, do not majority-vote fluent text. Compare claims, evidence, schema validity, and task-specific judgments.

A judge model is another fallible model. Evaluate position bias, verbosity bias, self-preference, prompt sensitivity, and correlated failure.

## Routing patterns

### Rule-based routing

Examples:

- image present → multimodal model;
- contract clause lookup → retrieval path;
- PHI → approved regional deployment;
- input exceeds context → hierarchical workflow;
- low-risk classification → small model.

Benefits: explainable and controllable.  
Risk: rules become brittle and difficult to maintain.

### Learned router

Predict which candidate is likely to satisfy quality at acceptable cost.

Inputs may include:

- task and modality;
- token/image/page counts;
- language;
- document quality;
- retrieval coverage;
- tenant/risk class;
- historical candidate performance;
- deadline and queue state.

Do not use protected or sensitive attributes merely because they improve routing. Review fairness and purpose.

### Similarity-based routing

Map a request to historical evaluated examples or task clusters.

Risk: similarity does not prove equivalent difficulty, authorization, or safety.

### Bandit or adaptive routing

Learn allocations from online outcomes.

Useful when outcomes arrive reliably and exploration is safe. Dangerous when labels are delayed, selective, or high-risk.

Constrain exploration to eligible, evaluated candidates and preserve a stable control. Do not let a bandit discover that unsafe behavior is “rewarding.”

## Router training data

The ideal routing dataset evaluates each request on every eligible candidate, producing counterfactual quality and cost labels.

In practice, this is expensive. Logged production data often shows only the chosen model’s outcome, causing selection bias.

Options:

- offline benchmark all candidates on a representative set;
- shadow candidates on sampled read-only traffic;
- randomized safe exploration;
- inverse-propensity or doubly robust analysis when assumptions hold;
- pairwise candidate comparisons;
- active sampling near route boundaries.

Log candidate eligibility, route probability, selected path, fallback, outcome, and cost.

Never train a router to reproduce the incumbent’s choices without measuring whether those choices were correct.

## Confidence is not certainty

Signals include:

- calibrated classifier probability;
- margin or entropy;
- ensemble disagreement;
- retrieval coverage;
- citation entailment;
- schema/business-rule validation;
- conformal set size;
- distance from known data;
- model self-report;
- consistency across perturbations;
- judge score;
- historical slice performance.

Model self-reported confidence is usually weak alone. A fluent explanation can accompany a wrong answer.

Combine signals in an evaluated acceptance policy. Preserve reason codes so operators know whether escalation came from missing evidence, disagreement, stale features, policy, or capacity.

## Calibration

A score is calibrated when predicted confidence matches observed correctness for a defined population and outcome.

Monitor calibration by:

- task;
- model/path;
- tenant or site;
- language;
- document quality;
- risk slice;
- time.

Calibration can drift when the input mix or model changes. A threshold of 0.9 is not portable between models, versions, or tasks.

Calibrate on data independent from training, using mature labels and representative routing conditions. If the router changes which cases reach a model, that model’s served population changes; recalibrate or monitor accordingly.

## Selective prediction and abstention

A selective system outputs only when confidence/evidence meets a threshold and abstains otherwise.

Key quantities:

- **coverage:** fraction of eligible cases answered automatically;
- **selective risk:** error rate among answered cases.

Plot risk versus coverage. The objective is not maximum coverage; it is useful coverage under a risk limit.

Abstention outcomes can be:

- ask the user for missing information;
- retrieve more evidence;
- run a stronger model;
- switch to an approved specialized path;
- return cited source material without synthesis;
- queue for human review;
- defer until a dependency recovers;
- decline the task.

An abstention must be truthful and operationally useful. “I cannot determine because drawing revision 4 is missing” is better than a generic refusal.

Abstention also has cost: review queues, user delay, and unequal coverage. Monitor who is being deferred.

## Conformal prediction

Conformal methods can produce prediction sets or intervals with coverage guarantees under stated exchangeability assumptions.

Examples:

- a set of possible document classes;
- an interval for a numeric quantity;
- a set of labels for an extraction field.

Benefits:

- explicit uncertainty set;
- finite-sample marginal coverage under assumptions;
- useful abstention trigger.

Limits:

- coverage may be marginal rather than per subgroup;
- distribution shift can break guarantees;
- generated free text is difficult to cover meaningfully;
- calibration data must match the deployed pipeline;
- large sets may be operationally useless.

Use conformal outputs as one policy signal, not a universal safety certificate.

## Disagreement handling

When models disagree:

1. compare evidence and source versions;
2. determine whether disagreement is resolvable by validation;
3. check whether one model is specialized for the slice;
4. retrieve missing evidence;
5. escalate if the consequence is high;
6. preserve both candidates and reasons for audit.

Do not choose the longest answer or the model with the highest self-confidence.

For invoice totals, arithmetic and source-region validation may resolve conflict. For contract interpretation, expert review may be required.

Disagreement itself is useful monitoring and active-learning data.

## Cost and latency accounting

Expected cascade cost:

[
E[C] = C_1 + P(escalate_1)C_2 + P(escalate_2)C_3 + cdots
]

Expected latency is more complex because of queueing, parallel work, caching, and deadlines.

Track:

- cost per request and correct outcome;
- stage exit rate;
- escalation rate;
- tail latency by route;
- duplicated work;
- queue delay;
- human handling;
- provider egress;
- cache effects;
- rejected/expired work.

A cascade can reduce average compute while worsening p99 latency for the hardest, highest-value cases. Reserve capacity for escalation paths.

Parallel ensembles multiply resource use. Use them selectively for cases where expected risk reduction justifies it.

## Capacity-aware routing

The router may consider queue age and deployment health, but policy remains primary.

If the best model is overloaded:

- queue within deadline;
- use an evaluated eligible fallback;
- narrow optional work;
- return evidence-only;
- route to human/manual workflow;
- reject with a truthful retry path.

Do not route PHI to an unapproved provider or allow a low-capability model to execute consequential tools merely to reduce queue length.

Capacity signals can create route oscillation. Use hysteresis, bounded changes, and stable load estimates.

## Multi-tenant fairness

One tenant’s difficult workload can consume every expensive-model slot.

Apply:

- per-tenant token and concurrency budgets;
- weighted fair queues;
- reserved capacity for critical classes;
- escalation quotas with aging;
- transparent overload behavior;
- tenant-specific eligibility and contracts;
- monitored abstention and quality by tenant.

Fairness is not identical allocation. A healthcare critical workflow may receive reserved capacity, but the policy must be explicit and governed.

Avoid a router that systematically sends small tenants or certain languages to weaker models because their short-term revenue or volume is lower.

## Safety and tool execution

Routing affects reasoning, not authorization.

Before any tool executes:

- validate complete structured output;
- recheck current authorization and policy;
- bind approval to exact operation and evidence versions;
- validate resource version;
- use idempotency;
- record selected path and lineage;
- reconcile outcome-unknown calls.

A small model may propose read-only search while only a stronger, separately evaluated path may propose a financial mutation. Even then, deterministic policy and approval decide execution.

Do not ensemble external writes. Multiple agents proposing the same action must collapse into one canonical operation before any side effect.

## Evaluation

Evaluate the whole policy, not only each candidate.

### Candidate quality

- task and critical-slice performance;
- calibration;
- latency and cost;
- privacy and region eligibility;
- schema and tool behavior.

### Router quality

- regret versus oracle candidate;
- route accuracy;
- cost under quality constraint;
- quality under budget;
- fairness and slice allocation;
- route stability;
- eligibility violations.

### Cascade quality

- stage exit and escalation;
- early-stop errors;
- tail latency;
- downstream correction;
- human queue load.

### Abstention quality

- risk-coverage curve;
- appropriate versus unnecessary abstention;
- missing-evidence reason accuracy;
- deferred-case outcomes;
- disparity in coverage.

### Failure evaluation

- model/provider outage;
- stale route metadata;
- capacity spike;
- calibration drift;
- malformed candidate output;
- retrieval failure;
- timeout between cascade stages;
- policy/revocation change;
- partial stream;
- uncertain external action.

Use fixed, current-production, challenge, and incident datasets. Shadow routing before user-visible rollout.

## Monitoring

In production, record:

- eligible candidate set and reasons;
- router/policy version;
- selected path and probability;
- stage outputs and validation;
- escalations and abstentions;
- actual resolved model/runtime;
- latency, cost, and capacity;
- final output/action;
- outcome and correction;
- fallback reason.

Monitor route share, eligibility failures, oracle gap on sampled shadows, calibration, risk-coverage, disagreement, capacity-driven fallback, tenant/language allocation, human queue load, and cost per correct outcome.

A stable aggregate accuracy can hide a router sending one slice to the wrong model.

## Construction operations example

A construction assistant processes invoices and change-order questions.

Routes:

1. deterministic extraction for native structured invoices;
2. small document model for clear scans;
3. multimodal model for complex tables or handwriting;
4. retrieval-plus-language model for contract/change-order support;
5. project-controls review for conflicting or consequential cases.

Policy:

- tenant/project authorization before routing;
- handwriting detector and layout validation;
- evidence coverage and source revision checks;
- calibrated field confidence;
- arithmetic reconciliation;
- stronger path for high-value invoices;
- no automatic ERP posting from any model;
- human approval bound to exact proposal.

If the small model extracts a total with high confidence but the line items do not reconcile, the cascade escalates. If the stronger model disagrees and the change order was approved after invoice submission, it abstains with a temporal-evidence reason.

At month-end, capacity-aware routing can defer low-priority batch summaries, but it cannot send tenant data to an unapproved provider or skip mandatory review.

## Healthcare operations variation

A prior-authorization packet-completeness assistant uses:

- deterministic FHIR/schema checks;
- OCR/layout model;
- regional PHI-approved multimodal model;
- payer-policy retrieval;
- trained operations review.

Hard filters enforce patient/organization/purpose, PHI region, task scope, and model approval. The system can identify missing documents but cannot infer medical necessity or deny care.

Abstention is mandatory for conflicting patient identity, illegible critical fields, unsupported payer policy, or missing clinical documentation. Monitor coverage and review burden by language, site, and approved fairness slices.

A smaller non-PHI provider is never a fallback for protected data. Capacity degradation returns the governed manual workflow.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Single general model | Simple operations | Overpay or underperform on slices |
| Rule router | Explainable | Brittle maintenance |
| Learned router | Adaptive efficiency | Selection bias and new failure mode |
| Sequential cascade | Cheap early exits | Tail latency and early-stop errors |
| Parallel ensemble | Error diversity | Multiplied cost |
| Judge model | Flexible comparison | Bias and correlated errors |
| High acceptance threshold | Lower selective risk | More abstention/review |
| High coverage | Automation value | More wrong answers |
| Deterministic validation | Strong invariants | Limited semantic coverage |
| Self-confidence | Cheap signal | Poor reliability alone |
| Disagreement escalation | Finds difficult cases | Higher cost |
| Capacity-aware routing | Availability | Quality oscillation |
| Tenant-specific routing | Correct constraints | Configuration complexity |
| Conformal sets | Explicit coverage target | Assumptions and broad sets |
| Human fallback | Accountability | Delay and scarce capacity |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Optimize cost before eligibility | Privacy/safety violation | Hard policy filter first |
| Router trained on incumbent choices | Repeats historical bias | Counterfactual candidate evaluation |
| Early model is overconfident | Cascade stops incorrectly | External validation and shadow oracle |
| Ensemble models share errors | False confidence | Measure diversity by slice |
| Judge favors verbosity | Wrong candidate selected | Blinded task-specific judge eval |
| Threshold copied across models | Miscalibrated decisions | Version/task calibration |
| Confidence from model statement | Fluent error accepted | Multi-signal acceptance |
| Route by sensitive proxy | Unequal quality/access | Purpose and fairness review |
| Capacity fallback untested | Outage becomes quality incident | Preapproved evaluated fallback |
| Router oscillates with load | Latency and cache instability | Hysteresis and bounded updates |
| One tenant consumes escalation | Other tenants starve | Fair queues and quotas |
| Abstention reason is generic | No recovery path | Structured reason/action |
| Human queue ignored | Hidden overload | Capacity-aware coverage policy |
| Disagreement resolved by longest text | Hallucination favored | Evidence/validation |
| Parallel agents execute tools | Duplicate side effect | Canonical proposal and one executor |
| Eligibility metadata stale | Revoked model still used | Versioned policy and fail closed |
| Router changes served population | Calibration drifts | Route-aware monitoring |
| Aggregate quality looks stable | Slice misrouting hidden | Allocation and outcome by slice |
| Timeout retries whole cascade | Duplicate cost/actions | Stage IDs and bounded replay |

## What to measure

Track candidate eligibility and rejection reasons; route share; routing regret against sampled oracle; stage exit/escalation; early-stop error; ensemble disagreement and error correlation; calibration; risk-coverage curve; abstention reason and outcome; human queue age; latency and cost by route; capacity-driven fallback; route stability; tenant/language/slice allocation; schema/citation/tool validity; unauthorized route attempts; correct outcomes; external effects; and cost per correct supported outcome.

The objective is not the cheapest average request. It is the lowest sustainable cost that meets quality, safety, fairness, and deadline constraints.

## Interview prompt

> Design a model-routing and abstention platform for a multi-tenant construction AI assistant. It handles invoice extraction, drawing questions, contract evidence, and ERP tool proposals using deterministic parsers, small models, large multimodal models, and human experts. Traffic is bursty and providers have different privacy, cost, and latency properties. Adapt it for healthcare prior-authorization operations with PHI.

Cover eligibility, routing, cascades, ensembles, confidence, calibration, selective prediction, capacity, multi-tenancy, tool safety, evaluation, monitoring, failure recovery, and healthcare constraints.

## Worked answer

**One-line conclusion:** Filter candidate paths by authorization, capability, residency, and risk; then choose the cheapest sufficient route using calibrated evidence, escalating or abstaining whenever the accepted-risk contract is not met.

“I would place a deterministic policy and capability filter before the router. It creates the eligible set from tenant/project authorization, task/modality, privacy/region, context, risk, model approval, health, and tool scope. Cost and latency optimize only within that set.

The cascade starts with native parsing, then a small specialized model, a larger multimodal or retrieval path, and finally expert review. Each stage emits a structured candidate, citations, validation, confidence features, and reason codes. The acceptance policy combines calibration, arithmetic/schema checks, retrieval coverage, disagreement, novelty, and task risk. A model’s self-confidence is only diagnostic.

For sampled traffic I shadow later stages to estimate early-stop oracle gap. Ensembles are reserved for high-value cases where measured error diversity justifies cost. Disagreement is resolved from source evidence or escalated, never by output length.

I evaluate the full policy with quality-under-budget, route regret, critical slices, risk-coverage, human load, tail latency, and fairness. Router training uses candidate benchmarking and safe randomized or shadow data, not only incumbent choices. Every decision logs eligible candidates, policy/router version, selected path, actual model, validation, abstention, cost, and outcome.

Capacity signals can queue or choose an evaluated fallback, but never weaken PHI, residency, or tool controls. Tool execution remains deterministic: current authorization, schema, resource version, approval binding, idempotency, and reconciliation.

For healthcare, only regional PHI-approved models are eligible; ambiguous identity, missing policy evidence, or critical illegibility abstains to trained staff, and no route may infer medical necessity or restrict care.”

## Practice lab

Design routing for a construction assistant when:

- 60% of invoices are native PDFs;
- 20% are clear scans;
- 10% contain handwriting or complex tables;
- 10% require contract/change-order interpretation;
- the small model is overconfident on handwritten totals;
- two large models agree often but share training data;
- the judge prefers longer answers;
- month-end exhausts multimodal capacity;
- one tenant requires regional processing;
- reviewers can handle only 40 cases per hour;
- a model proposes an ERP write before the cascade finishes;
- a provider revokes a model version mid-day.

Specify hard eligibility, stages, acceptance signals, calibration data, ensemble policy, risk-coverage target, capacity fallback, fairness, human-queue controls, tool boundary, logging, evaluation, and incident response.

Then adapt it to a prior-authorization completeness workflow with PHI, multilingual faxes, payer-policy retrieval, illegible patient identifiers, and a rule that automation cannot determine medical necessity or deny care.

## References

- [Selective Classification for Deep Neural Networks](https://arxiv.org/abs/1705.08500)
- [A Gentle Introduction to Conformal Prediction and Distribution-Free Uncertainty Quantification](https://arxiv.org/abs/2107.07511)
- [FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance](https://arxiv.org/abs/2305.05176)
- [RouteLLM: Learning to Route LLMs with Preference Data](https://arxiv.org/abs/2406.18665)
- [Mixture-of-Experts with Expert Choice Routing](https://research.google/pubs/mixture-of-experts-with-expert-choice-routing/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
