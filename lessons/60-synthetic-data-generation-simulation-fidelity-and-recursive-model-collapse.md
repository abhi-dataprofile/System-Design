# Synthetic-data generation, simulation fidelity, and recursive model collapse

Synthetic data is generated rather than directly observed from the target environment. It can expand rare scenarios, protect some sensitive attributes, create labels cheaply, test systems before deployment, and support simulation. It can also fabricate impossible combinations, copy private records, amplify generator bias, leak evaluation answers, and gradually replace reality with previous model output.

The central question is not “Does this sample look realistic?” It is:

> Does this generated dataset preserve the properties required for the decision, while remaining distinguishable, traceable, and bounded by real evidence?

## Mental model: synthetic data is a model output with a validity contract

Treat generation as a versioned transformation:

```text
real evidence + domain rules + generator + scenario plan
  -> synthetic candidate
  -> constraint, privacy, diversity, and fidelity checks
  -> approved use-specific dataset
  -> downstream model / evaluation / simulation
  -> real-world validation
```

Every synthetic record should answer:

- which generator, prompt, simulator, seed, and configuration produced it?
- which source records or distributions influenced it?
- what scenario and target property was requested?
- which hard constraints and review checks passed?
- where may it be used: training, testing, load, demonstration, or product output?
- how many synthetic generations separate it from directly observed data?

Synthetic origin is a first-class field, not metadata to remove before training.

## When to use synthetic data

Use synthetic data when it solves a specific shortage:

- rare but important safety or failure cases;
- early development before production data exists;
- controlled counterfactual or what-if scenarios;
- privacy-sensitive demonstrations and tests;
- class balancing after measuring real prevalence;
- simulator-generated sensor or operational data;
- structured-output, parser, and schema stress tests;
- adversarial and boundary-case evaluation;
- teacher-generated supervision with review.

Prefer real observed data when the goal depends on unknown human behavior, complex causal mechanisms, subtle tails, current prevalence, or legally authoritative outcomes.

Do not use synthetic examples to claim real-world effectiveness without evaluation on independent real data.

## Types of synthetic data

| Type | Example | Strength | Main risk |
|---|---|---|---|
| Rule or template generated | Valid invoices and JSON payloads | Exact control and coverage | Low behavioral realism |
| Simulation generated | Equipment movement or site sensor traces | Explicit dynamics and counterfactuals | Simulator-reality gap |
| Model generated | LLM-created RFIs or image variations | Broad, scalable variation | Hallucination and generator bias |
| Data augmentation | Crops, blur, noise, paraphrase | Preserves much source structure | Label may change under transformation |
| Statistical synthesis | Tabular records sampled from learned distribution | Marginal/joint similarity | Rare-record memorization and weak causality |
| Counterfactual generation | Change one factor while holding others | Tests sensitivity and fairness | Impossible or confounded scenarios |
| Procedural generation | Parametric drawings, layouts, scenes | Coverage and exact latent factors | Simplified world assumptions |

Keep these categories separate because their validation requirements differ.

## Define the validity contract

A synthetic dataset is valid only for a declared use.

Define:

- target task and decision;
- real population and deployment boundary;
- variables and causal assumptions;
- hard domain constraints;
- required marginal and joint distributions;
- rare scenarios and minimum coverage;
- time, tenant, geography, device, and workflow context;
- acceptable privacy and memorization risk;
- maximum synthetic proportion or curriculum by experiment;
- independent real-data evaluation gate;
- prohibited uses.

A dataset suitable for API load testing may be invalid for training invoice-risk predictions. A visually plausible site image may be invalid for estimating defect prevalence.

## Scenario design before generation

Write a scenario matrix rather than asking a generator for “more diverse examples.”

Dimensions could include:

- common versus rare condition;
- source quality and missingness;
- languages and terminology;
- project, vendor, facility, or device type;
- workflow stage;
- environmental conditions;
- adversarial or accidental corruption;
- protected and operational cohorts;
- valid, ambiguous, and invalid cases;
- abstention and escalation cases.

Specify correlations and exclusions. For example, a hospital renovation invoice, drawing revision, room state, and approval sequence must be mutually consistent. Independent sampling can create combinations that never occur.

Measure generated coverage against the scenario plan; do not infer diversity from a large row count.

## Generator architecture and isolation

A generation pipeline should:

1. resolve authorized source snapshots;
2. apply an approved scenario plan;
3. invoke a pinned generator or simulator;
4. capture prompts, seeds, parameters, and parent IDs;
5. run deterministic constraints;
6. run statistical, semantic, and privacy checks;
7. sample for domain review;
8. publish an immutable manifest.

Run untrusted generator code in a sandbox. Treat model-produced files, formulas, scripts, links, and tool calls as untrusted outputs. Generation must not submit external business actions.

Separate generation from evaluation. A generator that has access to protected answer keys can contaminate both training and tests.

## Fidelity is multidimensional

### Structural fidelity

Does the data obey schema, type, ranges, referential integrity, units, totals, and invariants?

Examples:

- invoice lines sum to the stated subtotal;
- tax and retainage follow declared rules;
- an RFI refers to an existing project and drawing revision;
- a room cannot be simultaneously occupied and available;
- timestamps follow the allowed workflow order.

### Statistical fidelity

Compare real and synthetic distributions:

- univariate marginals;
- correlations and conditional distributions;
- missingness patterns;
- sequence lengths and event intervals;
- class prevalence;
- cohort and tail coverage;
- temporal and geographic drift.

Matching marginal distributions is insufficient when joint relationships are wrong.

### Task fidelity

Train on synthetic and evaluate on independent real data. Compare:

- real-only baseline;
- synthetic-only;
- real plus synthetic;
- different mixture ratios;
- scenario or generator ablations.

A synthetic dataset is useful when it improves the target outcome or test coverage under real evaluation—not because a discriminator has difficulty separating it.

### Causal and intervention fidelity

If the use is decision support or simulation, test whether interventions have plausible effects. Correlation matching cannot validate counterfactuals.

Use domain equations, controlled experiments, historical natural experiments, or calibrated simulation where available. State unidentifiable assumptions.

### Operational fidelity

Generated data should exercise the real pipeline:

- file sizes and corruption modes;
- OCR and layout complexity;
- event ordering;
- tool constraints;
- user and reviewer behavior;
- queue and latency shape;
- region and permission boundaries.

## Simulation calibration and sim-to-real

A simulator encodes assumptions about the world. Calibrate parameters against real observations, then validate on held-out conditions.

Domain randomization deliberately varies textures, lighting, geometry, noise, or dynamics so a model does not overfit one simulator. Randomization ranges must remain physically and operationally plausible.

Use sensitivity analysis:

- which simulator parameters most affect downstream performance?
- do conclusions hold across plausible parameter ranges?
- where does the simulator fail to reproduce reality?
- which real observations would reduce uncertainty?

Do not tune the simulator only until it reproduces the evaluation set; that is another form of test overfitting.

## Privacy and memorization

Synthetic does not mean anonymous.

A generator may:

- reproduce a rare real record;
- preserve identifying combinations;
- emit names, addresses, account numbers, or PHI;
- reveal membership through repeated queries;
- allow linkage with external data;
- inherit licensed or restricted content.

Evaluate nearest-neighbor similarity, rare-record reproduction, membership and attribute inference risk, and explicit sensitive-pattern scans. Use data minimization, access controls, retention, and differential privacy where appropriate.

Differential privacy can provide a formal bound under stated assumptions, but it may reduce fidelity. Record the privacy unit, parameters, accountant, and composition. Do not label a dataset “privacy safe” based only on de-identification or visual inspection.

## Synthetic labels and teacher models

A teacher model can propose labels, rationales, preferences, or examples. Preserve:

- teacher and prompt version;
- source evidence available;
- confidence and sampling policy;
- verification and adjudication status;
- whether the label was accepted, corrected, or rejected.

Teacher agreement is not truth. Multiple teachers can share training data and systematic errors. Use trusted labels, domain review, deterministic validators, and disagreement sampling.

Do not train on model-generated reasoning as if it were an auditable factual chain. Evaluate the student on outcomes and constraints.

## Recursive model collapse

Recursive training occurs when models are trained on data generated by earlier models, directly or through polluted public sources. Repeated generations can:

- underrepresent distribution tails;
- amplify common modes;
- erase rare language or behaviors;
- increase artifacts and errors;
- make successive datasets appear smoother but less faithful;
- entrench previous model bias.

Track **synthetic depth**:

- depth 0: directly observed or authoritative record;
- depth 1: generated from depth-0 evidence;
- depth 2+: generated from outputs already containing synthetic influence.

Controls include:

- retain a substantial, versioned real-data anchor;
- cap synthetic influence by task, cohort, and depth;
- prevent generated outputs from silently re-entering raw-data stores;
- detect model-origin markers and lineage;
- refresh from new real observations;
- evaluate every generation on unchanged real holdouts and tails;
- monitor distribution entropy, coverage, and rare-mode recall;
- quarantine unknown-origin web or feedback data when provenance is material.

There is no universal safe percentage. Determine mixture experimentally and report it.

## Evaluation contamination

Synthetic generation can leak protected evaluation material through:

- prompts containing benchmark questions;
- teacher access to answer keys;
- retrieval over evaluation sources;
- generated paraphrases of test items;
- production prompt logs later used for training;
- evaluator explanations copied into examples.

Scan synthetic outputs and parent prompts against the full benchmark family: question, answer, explanation, source, translation, and paraphrase. Maintain separate generator credentials and retrieval indexes for training and protected evaluation.

## Bias and representation

Generation often improves counts while worsening representation. Asking for equal examples can erase real prevalence; asking a model for demographic diversity can create stereotypes or impossible correlations.

Report:

- real and synthetic counts separately;
- mixture weights;
- generator acceptance rates;
- constraint failures by cohort;
- downstream performance by real cohort;
- rare-case recall;
- reviewer disagreement.

Synthetic examples should supplement missing coverage, not stand in for engagement with affected populations or real operational evidence.

## Release and monitoring

Publish a synthetic dataset as a signed manifest containing:

- scenario plan and intended use;
- source and parent lineage;
- generator/simulator and configuration;
- prompts, seeds, versions, and synthetic depth;
- validation code and results;
- privacy analysis;
- review samples and decisions;
- mixture and weighting policy;
- prohibited uses;
- known simulator or generator gaps.

Monitor whether production performance differs specifically on synthetic-heavy scenarios. If failures cluster there, reduce influence, revise the simulator, or collect real evidence.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Rule-based generation | Exact constraints and labels | Narrow, unrealistic variation |
| Model-based generation | Scalable semantic variety | Hallucination, bias, memorization |
| Simulation | Controlled interventions | Reality gap and calibration cost |
| More synthetic data | Coverage and lower marginal cost | Correlated errors and false sample size |
| Aggressive filtering | Higher apparent quality | Removes difficult and rare cases |
| Human review | Domain validity | Expensive, sampled, and inconsistent |
| Differential privacy | Formal privacy bound | Utility and tail-fidelity loss |
| Synthetic-only training | Low raw-data exposure | Weak grounding and collapse risk |
| Real-plus-synthetic mix | Coverage with anchor | Mixture tuning and lineage complexity |
| Domain randomization | Robustness across variation | Implausible samples if ranges are wrong |

## Failure modes and defenses

| Failure mode | Defense |
|---|---|
| Generated invoices do not sum correctly | Deterministic accounting constraints |
| Simulator matches averages but misses tails | Conditional, tail, and scenario coverage tests |
| Synthetic-only model excels on synthetic test | Independent real holdout and TSTR-style evaluation |
| Generator repeats a rare customer record | Nearest-neighbor, memorization, and sensitive-pattern checks |
| Teacher hallucinates labels | Evidence-bound proposals, trusted review, and disagreement sampling |
| Filtering removes ambiguous examples | Track rejection by cohort and retain explicit ambiguity scenarios |
| Synthetic variants leak across train and test | Parent-family grouping and lineage-safe splitting |
| Protected benchmark enters teacher prompt | Isolated evaluation boundary and contamination scan |
| Model output returns as “real” feedback | Origin labels, write-path separation, and synthetic-depth tracking |
| Repeated generations erase rare modes | Real-data anchor, depth caps, entropy and tail monitoring |
| Privacy claim rests on removing names | Inference-risk testing and formal privacy mechanism where needed |
| Domain randomization creates impossible scenes | Physics/business constraints and domain review |
| Synthetic balance distorts real prevalence | Separate coverage training from calibrated deployment evaluation |

## Construction example: invoice and safety-inspection data

A construction AI team lacks examples of rare invoice exceptions and low-light firestopping defects.

For invoices, rule-based generation creates line items, taxes, retainage, change-order references, and approval histories under deterministic accounting constraints. A language model varies descriptions but cannot change authoritative numeric fields. Every record links to a scenario, generator, prompt, seed, and synthetic depth.

For inspection images, a simulator varies lighting, dust, camera angle, occlusion, wall type, and defect geometry within ranges calibrated from real site photos. Domain experts review samples from each rare scenario. The team compares real-only, synthetic-only, and mixed training on untouched real projects.

The mixed model improves rare-defect recall, but one synthetic texture causes false positives. An ablation identifies the simulator artifact, and the affected generator version is revoked. Generated images never enter the “observed field evidence” store.

Invoice teacher labels are checked against deterministic totals and sampled project-manager review. A public benchmark is excluded from teacher retrieval, and synthetic paraphrases are grouped with their parents before splitting.

A healthcare-operations version may simulate scheduling, room turnover, or equipment flows, but patient behavior and clinical outcomes require stronger causal validation, privacy controls, and real-world evaluation. Synthetic records must never be mistaken for clinical evidence.

## Interview prompt

**Design a synthetic-data platform for a construction AI system that needs rare invoice exceptions, RFI cases, and safety-inspection images. Protect sensitive records, validate simulation fidelity, prevent train/test leakage, and avoid recursive model collapse as generated data is reused over time.**

## Worked answer

1. **Clarify the shortage and claim.** Define the task, target population, rare scenarios, real-data limits, privacy unit, downstream decision, and required evidence.
2. **Choose generation by structure.** Use rules for exact accounting and schemas, simulation for physical scenes, augmentation for bounded transformations, and models for language variation under constraints.
3. **Create a scenario contract.** Specify correlations, exclusions, tails, cohorts, ambiguity, adversarial cases, and permitted uses before generating.
4. **Capture lineage.** Store source parents, generator, prompt, seed, configuration, synthetic depth, rights, purpose, and validation outcomes.
5. **Validate at several levels.** Enforce hard constraints; compare marginal, joint, temporal, cohort, and tail distributions; calibrate simulations; and review stratified samples.
6. **Test real utility.** Compare real-only, synthetic-only, and mixture ratios on independent real temporal and project holdouts with generator ablations.
7. **Protect privacy and evaluation.** Scan memorization and sensitive values, apply formal privacy where justified, isolate benchmark material, and group synthetic children with parents.
8. **Control recursive reuse.** Separate generated from observed stores, cap depth and influence, retain real anchors, refresh from new real data, and monitor entropy and rare-mode coverage.
9. **Release deliberately.** Publish signed manifests with intended use, mixture policy, constraints, privacy evidence, gaps, and revocation support.
10. **Monitor reality gaps.** Slice production errors by synthetic-heavy scenario; revoke bad generators, collect real examples, recalibrate, and retrain through normal release gates.

**One-line conclusion:** Synthetic data is valuable only when its origin, assumptions, fidelity, privacy, and generation depth remain visible—and every claimed improvement survives evaluation on independent real-world evidence.

## References

- [The Curse of Recursion: Training on Generated Data Makes Models Forget](https://www.nature.com/articles/s41586-024-07566-y)
- [Model Autophagy Disorder in Generated Data](https://arxiv.org/abs/2307.01850)
- [Domain Randomization for Transferring Deep Neural Networks from Simulation to the Real World](https://arxiv.org/abs/1703.06907)
- [Synthetic Data — Anonymisation Groundhog Day](https://www.usenix.org/conference/usenixsecurity23/presentation/stadler)
- [Data Cards: Purposeful and Transparent Dataset Documentation for Responsible AI](https://dl.acm.org/doi/10.1145/3531146.3533231)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
