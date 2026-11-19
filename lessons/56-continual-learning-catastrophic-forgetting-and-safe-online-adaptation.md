# Continual learning, catastrophic forgetting, and safe online adaptation

A production model operates in a changing world. Vendors, document formats, equipment, project phases, policies, users, and workflows evolve. Updating the model can recover performance on new conditions, but it can also erase old capabilities, amplify biased feedback, learn from its own mistakes, or adapt to an attack.

Continual learning is therefore a controlled release process over a time-varying data stream—not permission for a production model to rewrite itself after every interaction.

## Mental model: observe continuously, learn in candidates, promote deliberately

Use two separated loops:

```text
production loop:
input -> immutable release -> prediction -> review/action -> delayed outcome

learning loop:
versioned events -> label maturation -> sampling + replay
 -> candidate training -> temporal/cohort/forgetting evaluation
 -> approval -> shadow/canary -> new immutable release
```

Maintain three distinct states:

1. **Production model:** immutable, identified, reproducible, and rollbackable.
2. **Candidate learner:** allowed to adapt in an isolated training environment.
3. **Learning state:** replay buffer, optimizer state, statistics, privacy ledger, and curriculum—versioned separately from model weights.

“Online data arrival” does not require “online production updates.” Most business systems should collect evidence continuously but promote models in bounded batches.

## When to use continual learning

Consider continual learning when:

- meaningful drift occurs faster than occasional full retraining can handle;
- labels or outcomes arrive repeatedly after deployment;
- new classes, vendors, layouts, devices, or operating conditions appear;
- personalization creates durable value;
- retaining historical performance is measurable;
- the team can operate strong evaluation, lineage, and rollback controls.

Prefer periodic retraining when changes are slow, labels are delayed or unreliable, model updates are expensive, governance requires explicit review, or a stable baseline is more valuable than rapid adaptation.

Prefer retrieval, rules, configuration, or prompt changes when the new information is factual or policy-driven rather than a learned pattern. Do not train a model merely to memorize a newly issued contract clause.

## Update modes

| Mode | Behavior | Appropriate use |
|---|---|---|
| Full retraining | Rebuild from curated historical and recent data | Strong reproducibility; slower and expensive |
| Incremental batch | Update from recent data plus replay on a schedule | Common production default |
| Streaming/online optimization | Update after small batches or events | Fast drift with mature safeguards |
| Adapter or head update | Freeze base; learn a limited component | Safer personalization or new labels |
| Ensemble of experts | Add specialist models and route | Distinct regimes or cohorts |
| Instance memory/retrieval | Store approved examples without changing weights | New facts, auditable corrections |
| Rule/configuration update | Change deterministic policy | Compliance and workflow changes |

The safest architecture often uses several: retrieval for fresh facts, rules for policy, an adapter for tenant-specific patterns, and slower base-model updates.

## Event and label lineage

Every learning example should connect:

- immutable input and source revision;
- production prediction and release ID;
- decision context and available evidence;
- reviewer action and expertise;
- eventual operational outcome;
- label definition and guideline version;
- selection reason and sampling probability;
- tenant, cohort, time, purpose, consent, and retention;
- whether the model influenced the label or outcome.

Delayed labels must use event time and knowledge time. An invoice marked “paid” today may have been overdue when the prediction was made. Reconstruct features as they existed at decision time.

Do not train directly from clicks, approvals, or edits without modeling what they mean. Users may accept suggestions because they are busy, correct only severe cases, or never see abstained examples.

## Feedback loops and selective labels

Production predictions change the data later observed:

- flagged invoices receive expert review, while unflagged invoices do not;
- recommended actions alter outcomes;
- high-confidence automation removes examples from human labeling;
- reviewers anchor on model suggestions;
- active learning oversamples uncertain or rare cases.

This creates selective labels and treatment effects. Store the selection policy and probability. Maintain exploration or randomized review where safe, and use inverse-propensity or causal methods only when their assumptions are defensible.

Keep a model-independent audit sample. Without it, the platform may “improve” only on cases it chose to observe.

## Catastrophic forgetting

Catastrophic forgetting occurs when learning new patterns degrades previously learned behavior.

Measure a task/cohort performance matrix:

[
M_{i,j} = 	ext{performance on evaluation cohort } j 	ext{ after update } i
]

Useful release metrics include:

- current-cohort gain;
- worst historical-cohort regression;
- average forgetting from each cohort’s best prior score;
- backward transfer to old cohorts;
- forward transfer to unseen or emerging cohorts;
- calibration and abstention changes;
- safety and policy invariant failures.

Never compress this into one average. A new model can improve overall while forgetting handwritten invoices, a minority language, a small tenant, or a safety-critical condition.

## Mitigation strategies

### Rehearsal and replay

Train on recent examples plus a representative historical buffer. Preserve rare, high-impact, boundary, and adversarial examples.

Replay buffers need:

- source and label lineage;
- tenant and purpose authorization;
- retention and deletion propagation;
- deduplication and leakage controls;
- balanced cohort policy;
- protected evaluation separation;
- refresh rules when label definitions change.

A random reservoir may preserve frequency but lose rare safety cases. Stratified or priority replay improves coverage but can distort prevalence; evaluation must use realistic distributions too.

### Regularization

Methods such as elastic weight consolidation penalize changes to parameters important for previous tasks. They can reduce forgetting but depend on importance estimates and task structure. They are not a replacement for replay and evaluation.

### Architectural isolation

Freeze a stable base and learn adapters, heads, experts, or tenant-specific modules. This bounds some regressions and simplifies rollback, but increases routing, storage, versioning, and support complexity.

### Distillation

Train the candidate to preserve selected behavior from the prior model while learning new data. Distillation can also preserve old errors or hide regression outside the teacher’s observed domain.

### Dynamic models

Add capacity for new tasks or regimes. This can reduce interference, but unbounded growth makes serving and governance difficult. Define merging, retirement, and routing policies.

## Drift trigger and update policy

A drift alarm should not automatically train or deploy a model.

A trigger policy may combine:

- input and feature distribution changes;
- prediction and uncertainty changes;
- delayed outcome degradation;
- reviewer correction and escalation rates;
- new-class volume;
- business or policy change;
- minimum label maturity and sample size;
- confidence that the signal is not a pipeline incident.

The response could be investigate, adjust monitoring, update retrieval, relabel data, train a candidate, or roll back. Diagnosis precedes adaptation.

Use cooldowns and minimum evidence so noisy metrics do not create update thrashing.

## Candidate training pipeline

A candidate run should pin:

- base release and learning-state versions;
- recent-data window and label-maturity cutoff;
- replay-buffer snapshot and sampling weights;
- feature, preprocessing, ontology, and policy versions;
- optimizer, hyperparameters, seeds, and update strategy;
- excluded tenants, regions, or revoked data;
- privacy budget where applicable;
- training code and runtime;
- intended deployment cohorts.

Prevent training-serving skew. If online feature logic changes, evaluate the complete release bundle rather than only the weights.

## Evaluation gates

Use multiple suites:

1. **Recent temporal holdout:** measures current adaptation.
2. **Historical replay suites:** detect forgetting across periods and regimes.
3. **Stable gold set:** detects broad regressions; protect it from repeated tuning.
4. **Rare and safety-critical set:** enforces hard invariants.
5. **Tenant and cohort slices:** prevents average metrics from hiding harm.
6. **Counterfactual or challenge set:** tests shortcuts and spurious drift responses.
7. **Shadow production:** tests real inputs without influencing outcomes.

Set both improvement and non-regression gates. Use confidence intervals and minimum sample sizes. Some metrics should be hard blockers, such as cross-tenant leakage, invalid tool actions, or regulated-policy violations.

## Promotion and rollback

Promote candidates as immutable releases through:

- offline approval;
- shadow comparison;
- limited canary;
- staged cohort or regional rollout;
- enhanced monitoring;
- explicit expansion gates.

Keep learning state separate so rolling back serving does not accidentally continue from a contaminated optimizer or replay snapshot.

If a release fails, roll back the complete compatible bundle and quarantine the candidate’s model, optimizer, replay mutations, derived features, and generated labels until investigated.

Never let production traffic update weights in place without an atomic release boundary.

## Data deletion and model influence

When a user or tenant requests deletion, remove eligible records from source datasets, replay buffers, feature snapshots, caches, and future training.

Whether the trained model must be retrained or unlearned depends on policy, law, threat model, and feasible assurance. Record which releases and candidates used the data. Do not claim that deleting a row from the replay buffer removes its influence from existing weights.

## Security threats

An adversary can exploit adaptation through:

- repeated poisoned feedback;
- sybil users or devices;
- rare-trigger backdoors;
- label-flipping;
- malicious reviewer corrections;
- data flooding that changes class balance;
- gradual “boiling frog” drift below per-window thresholds;
- model-generated synthetic examples fed back as truth.

Use identity, rate limits, provenance, reviewer trust boundaries, influence caps, temporal anomaly detection, adversarial regressions, and manual review for high-impact changes. Separate generated data from observed outcomes.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Frequent updates | Faster adaptation | Noise, forgetting, release burden, attack surface |
| Periodic batches | Better governance and reproducibility | Slower response to drift |
| Large replay buffer | Better historical retention | Storage, privacy, stale labels, training cost |
| Small recent window | Fast adaptation | Severe forgetting and seasonal bias |
| Frozen base plus adapters | Bounded change and easier rollback | Routing and artifact complexity |
| Full-model updates | Maximum flexibility | Larger regression surface |
| Model-independent audit sample | Unbiased visibility | Manual-review cost |
| Train from user behavior | Abundant feedback | Selection bias, anchoring, and manipulation |
| Automatic promotion | Fast iteration | Unsafe under weak evaluation |
| Human approval | Contextual judgment | Slower cadence and reviewer bottlenecks |

## Failure modes and defenses

| Failure mode | Defense |
|---|---|
| New vendor format improves while old formats collapse | Historical cohort matrix and non-regression gates |
| Only flagged cases receive labels | Random audit sample and stored selection propensity |
| Reviewer accepts model suggestions automatically | Blind or independent review samples and anchoring analysis |
| Drift alarm triggers endless retraining | Diagnosis gate, minimum evidence, cooldown, and update budget |
| Replay buffer violates deletion request | End-to-end lineage and deletion propagation |
| Recent duplicates dominate training | Entity/time-aware deduplication and sampling caps |
| Policy change is learned as correlation | Implement policy deterministically; do not infer it from outcomes |
| Poisoned feedback gradually shifts model | Provenance, influence limits, temporal anomaly detection, adversarial tests |
| Rollback restores weights but not optimizer state | Version and quarantine complete learning state |
| Online update changes weights mid-request | Immutable serving releases and atomic routing switch |
| Average improves but small tenant regresses | Tenant/cohort worst-case gates |
| Synthetic labels recursively become truth | Explicit origin labels and limits on generated-data reuse |
| Stale labels train against future knowledge | Event-time reconstruction and label-maturity cutoff |

## Construction example: evolving invoice exception detection

A construction company predicts which invoices need project-manager review. New subcontractors, OCR layouts, cost codes, and project phases appear throughout the year. Payment and dispute outcomes arrive weeks later.

The production model remains immutable. Each prediction records the invoice revision, features, model release, review selection, reviewer action, and later ERP outcome. A random audit sample includes low-risk invoices that the model would otherwise never send to review.

Every month, the learning pipeline creates a candidate using matured recent outcomes plus a stratified replay buffer covering prior vendors, handwritten scans, high-value change orders, winter work, and rare duplicate-payment cases. It reconstructs features as known at invoice time.

The candidate improves new-vendor recall but forgets a legacy handwritten format. The historical cohort gate rejects it. The next candidate uses additional replay and a format-specific adapter, passes temporal and forgetting tests, and enters shadow mode.

A burst of nearly identical “corrections” from one account is capped and investigated rather than immediately learned. A changed approval policy is implemented in the policy engine, not inferred from recent labels.

A healthcare-operations version could adapt a room-turnover or scheduling model, but must account for delayed outcomes, treatment effects, site differences, protected cohorts, and clinical boundaries. No model should adapt directly from patient-care actions without approved governance.

## Interview prompt

**Design a continual-learning system for construction invoice exception detection. Vendor formats and project conditions change, outcomes arrive weeks later, only some invoices receive review, tenants differ, old capabilities must not regress, and poisoned feedback must not reach production.**

## Worked answer

1. **Clarify adaptation need.** Define drift speed, label delay, update cadence, tenants, safety invariants, acceptable regression, and rollback objective.
2. **Separate loops.** Keep production releases immutable; collect versioned evidence continuously; train candidates only in an isolated learning pipeline.
3. **Build lineage.** Join source revision, prediction, review selection, correction, and delayed outcome using stable IDs and event-time features.
4. **Correct observation bias.** Record selection policies, maintain a model-independent audit sample, and distinguish reviewer behavior from ground truth.
5. **Manage memory.** Combine matured recent data with authorized, deduplicated, cohort-aware replay; preserve rare and adversarial cases while honoring deletion.
6. **Choose adaptation scope.** Prefer adapters or heads for bounded changes; use full retraining when representations truly need updating; use rules or retrieval for policy and facts.
7. **Evaluate forgetting.** Run recent temporal, historical cohort, stable gold, rare-safety, tenant, calibration, and adversarial suites with improvement and non-regression gates.
8. **Defend feedback.** Require provenance, cap influence, detect sybils and repeated corrections, separate synthetic labels, and quarantine suspicious windows.
9. **Release progressively.** Register the full candidate and learning state, then shadow, canary, stage, monitor, and expand through explicit gates.
10. **Recover completely.** Atomically restore the prior release bundle and quarantine candidate weights, optimizer, replay mutations, and derived labels; diagnose before the next update.

**One-line conclusion:** Safe continual learning adapts through versioned candidates and representative memory while immutable production releases, forgetting gates, feedback controls, and complete rollback prevent the model from learning faster than the organization can verify.

## References

- [Overcoming catastrophic forgetting in neural networks](https://www.pnas.org/doi/10.1073/pnas.1611835114)
- [Gradient Episodic Memory for Continual Learning](https://proceedings.neurips.cc/paper/2017/hash/f87522788a2be2d171666752f97ddebb-Abstract.html)
- [Continual Lifelong Learning with Neural Networks: A Review](https://www.sciencedirect.com/science/article/pii/S0893608019300231)
- [River: online machine learning documentation](https://riverml.xyz/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [Google Rules of Machine Learning](https://developers.google.com/machine-learning/guides/rules-of-ml)
