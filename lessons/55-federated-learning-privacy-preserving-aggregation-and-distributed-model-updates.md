# Federated learning, privacy-preserving aggregation, and distributed model updates

Federated learning coordinates model training across organizations or devices while keeping raw training records at their source. It can reduce central data collection, but it does not automatically provide privacy, security, fairness, or regulatory compliance.

Model updates can reveal information, participants can poison training, and heterogeneous local data can make a mathematically valid average perform badly. The system must govern the complete round—from participant eligibility through aggregation, evaluation, release, and deletion obligations.

## Mental model: distributed computation with untrusted contributions

A federated round is a versioned protocol:

```text
coordinator selects eligible clients
  -> sends signed training plan and base model
  -> clients validate, train locally, clip and protect updates
  -> secure aggregation combines accepted contributions
  -> server evaluates candidate on independent suites
  -> registry records lineage
  -> approved release rolls out with rollback
```

Separate five artifacts:

1. **Training plan:** model, optimizer, local steps, features, clipping, privacy, and eligibility rules.
2. **Local computation:** data selection and training performed inside a participant boundary.
3. **Contribution:** update plus protocol metadata—not a raw dataset.
4. **Aggregate candidate:** combined update before production approval.
5. **Released model:** an evaluated, signed artifact with complete round lineage.

Federated learning changes where computation happens. It does not remove the need for data governance or release controls.

## When to use federated learning

Consider federated learning when:

- organizations cannot or should not centralize raw records;
- data is naturally partitioned across hospitals, contractors, facilities, or edge fleets;
- local data has useful variation that cannot be reproduced centrally;
- participants can operate a compatible, attested training runtime;
- the expected quality gain justifies protocol and operational complexity.

Prefer centralized training when data can lawfully and securely be pooled, participants are few and trusted, local compute is unreliable, reproducibility is more important than locality, or a representative central dataset already solves the problem.

Use alternatives when appropriate:

- federated analytics for aggregate statistics without model training;
- split learning when model stages, rather than examples, are divided;
- confidential computing for protected centralized computation;
- transfer learning with local fine-tuning when a shared global update adds little value;
- retrieval or rules when the requirement is local knowledge, not learned parameters.

## Cross-device versus cross-silo

| Property | Cross-device | Cross-silo |
|---|---|---|
| Participants | Many phones, sensors, or edge devices | Fewer organizations or sites |
| Availability | Intermittent and sampled | Scheduled and more predictable |
| Trust | Devices may be compromised | Organizations are known but not fully trusted |
| Data volume | Small per device | Larger per organization |
| Dropout | Expected within every round | Operational incident or policy event |
| Identity | Device and user eligibility | Contract, organization, workload, and jurisdiction |
| Typical aggregation | Large anonymous cohorts | Smaller governed cohorts |
| Main concern | Scale, dropout, battery, privacy | Governance, heterogeneity, attribution, legal boundaries |

Do not use a cross-device threat model for a five-hospital consortium. A small cohort makes individual updates easier to infer and each participant more influential.

## Round coordinator and state machine

Represent every round with explicit states:

```text
planned -> enrolling -> training -> collecting -> aggregating
        -> evaluating -> approved / rejected / aborted
```

A round manifest should identify:

- round and experiment IDs;
- base model and training-code hashes;
- eligible participant set and selection policy;
- feature and label contracts;
- local data window and exclusion rules;
- optimizer, learning rate, local epochs, and random seeds where applicable;
- update clipping and weighting rules;
- secure-aggregation threshold;
- differential-privacy parameters and accountant state;
- minimum and maximum cohort sizes;
- timeout, dropout, retry, and abort policy;
- evaluation gates and release owner.

The coordinator must be idempotent. A participant retry must not count twice, and an aborted round must never be published accidentally.

## Client eligibility and local execution

Before each round, verify:

- organization, device, and workload identity;
- software and training-plan signature;
- model and schema compatibility;
- approved region, purpose, and data window;
- local consent or legal basis where required;
- device posture and resource limits;
- prior privacy-budget usage;
- participant-specific exclusion or revocation.

Run training in a restricted environment. The plan should expose only declared features and aggregates, not arbitrary queries over the participant’s data. Limit filesystem, network, runtime, and output size. Sign contributions with a round-bound workload identity.

Local validation should reject corrupted data, impossible labels, unsupported schema, and training instability. It should report bounded health metadata without leaking raw records.

## Non-IID data and weighting

Federated data is rarely independent and identically distributed:

- a contractor may specialize in one trade;
- one hospital serves a different population;
- devices have different sensors;
- labels follow local operational practices;
- participation depends on connectivity and resources.

Federated averaging typically weights local updates by example count, but that may let one large organization dominate and may optimize average performance while harming smaller cohorts.

Evaluate weighting choices:

- example-count weighting;
- equal participant weighting;
- capped influence;
- quality- or freshness-aware weighting;
- stratified aggregation;
- personalized heads or local adapters;
- clustered models for distinct populations.

Test the global model and each meaningful participant cohort. Report distribution, worst-group, and change-from-baseline metrics. A global average can improve while an important hospital or subcontractor regresses.

## Secure aggregation

Secure aggregation lets the server recover an aggregate without seeing each participant’s plaintext update, assuming protocol conditions hold.

It protects against a curious coordinator reading individual updates, but it does not by itself:

- make malicious updates safe;
- prevent inference from the final model;
- hide participant membership from every observer;
- protect against colluding participants beyond the protocol’s assumptions;
- solve small-cohort disclosure;
- validate that local training used approved data;
- provide differential privacy.

Define the threshold and dropout behavior explicitly. If too few participants remain, abort rather than silently switching to plaintext or a weaker mode. Separate protocol operators and key material where the threat model requires it.

Secure aggregation complicates anomaly detection because individual updates are hidden. Use bounded updates, client-side validation, zero-knowledge or attestation techniques where justified, multiple aggregates, trusted execution, and post-aggregation evaluation rather than assuming inspection is always possible.

## Differential privacy

Differential privacy can bound the influence of an individual record or participant, depending on the mechanism and unit of privacy.

A typical training mechanism:

1. clips each contribution to a maximum norm;
2. aggregates clipped contributions;
3. adds calibrated random noise;
4. tracks cumulative privacy loss with an accountant.

State clearly:

- privacy unit: record, user, device, or organization;
- sampling assumptions;
- clipping norm;
- noise mechanism;
- epsilon, delta, and accounting method;
- number of rounds and composition;
- stopping rule when budget is exhausted.

Privacy parameters are not meaningful without these assumptions. Do not market “DP enabled” without an interpretable guarantee and measured utility.

Participant-level privacy in a small cross-silo consortium may require enough noise to damage utility. Governance, contractual controls, secure aggregation, and restricted release may remain necessary.

## Poisoning and Byzantine behavior

A participant can submit an update that:

- degrades global accuracy;
- implants a backdoor;
- targets one cohort;
- cancels another update;
- exploits weighting;
- causes numeric overflow or denial of service.

Defenses include:

- round-bound identity and one accepted update per participant;
- norm clipping and numeric validation;
- influence caps;
- robust aggregation such as coordinate-wise median or trimmed methods when assumptions fit;
- canary evaluation on clean and adversarial suites;
- backdoor and targeted-cohort tests;
- participant reputation used cautiously;
- quarantine and investigation of suspicious rounds;
- rollback and revocation.

Robust aggregation has tradeoffs and may fail in high dimensions or under non-IID honest data. Treat it as one layer, not proof of safety.

## Communication, dropout, and stale updates

Updates can be large and participants unreliable. Use compression, quantization, sparse updates, checkpointing, and scheduled windows only after measuring quality effects.

Bind every contribution to an exact base-model hash and round. Never apply a stale update to a newer model without an explicit algorithm that supports staleness.

Set deadlines and thresholds. If a participant disconnects:

- resume only with a round-bound token and checkpoint;
- do not duplicate its contribution;
- remove or reconstruct secure-aggregation masks according to the protocol;
- abort if privacy or quorum assumptions fail;
- record whether the participant’s privacy budget was consumed.

Do not keep extending a round until only the fastest, best-connected participants determine the model. That creates participation bias.

## Evaluation and release

Never promote the aggregate directly.

Evaluation should include:

- central or jointly governed holdout data;
- participant-local evaluation with privacy-safe summaries;
- worst-cohort and leave-one-silo-out analysis;
- calibration, fairness, safety, and backdoor tests;
- comparison with base and previous production models;
- robustness to missing participants;
- privacy and secure-aggregation audit;
- inference cost and operational compatibility.

Register the candidate with the complete manifest, participant eligibility set, contribution receipts, aggregation code, privacy accountant, evaluation evidence, and approvals.

Roll out through shadow, canary, and staged deployment. Retain a known-good model and the ability to stop future rounds. A participant may revoke future use, but removing historical influence from a trained model is a separate, difficult problem requiring policy and potentially retraining or machine unlearning.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Central training | Simpler debugging and reproducibility | Raw-data centralization and larger data blast radius |
| Federated training | Data locality and cross-silo learning | Protocol, heterogeneity, and security complexity |
| Secure aggregation | Hides individual plaintext updates from coordinator | Threshold, dropout, and anomaly-detection complexity |
| Differential privacy | Formal influence bound under stated assumptions | Utility loss and budget management |
| More local epochs | Less communication | Client drift and stronger local bias |
| Example-count weighting | Matches total-record objective | Large participants dominate |
| Equal participant weighting | Limits size dominance | Small noisy clients gain influence |
| Personalized models | Better local fit | More artifacts, evaluation, and support |
| Aggressive compression | Lower bandwidth | Accuracy loss and new numerical behavior |
| Small cohorts | Easier scheduling | Weaker privacy and greater individual influence |

## Failure modes and defenses

| Failure mode | Defense |
|---|---|
| Coordinator claims raw data never leaves, but gradients leak | Secure aggregation, clipping, DP where appropriate, and honest threat communication |
| One large participant dominates | Influence caps, alternative weighting, and cohort metrics |
| Honest non-IID updates look malicious | Domain-aware validation and robust methods tested on realistic heterogeneity |
| Malicious client installs a backdoor | Identity, clipping, influence limits, adversarial evaluation, quarantine, rollback |
| Dropout breaks secure aggregation | Threshold protocol, mask recovery, explicit abort conditions |
| Stale update reaches a newer model | Base-model hash and round-bound contribution |
| Retry counts one participant twice | Stable participant-round ID and idempotent receipt |
| Fast clients bias every round | Availability-aware sampling, participation tracking, and delayed windows |
| Privacy budget is reset by a new job | Central accountant keyed to privacy unit and purpose |
| Small cohort falls back to plaintext | Fail closed and abort; never silently weaken protocol |
| Aggregate passes average metrics but harms one hospital | Per-cohort and worst-group release gates |
| Participant withdraws after release | Prospective exclusion plus documented retraining/unlearning policy |
| Local runtime runs arbitrary coordinator code | Signed declarative plan, sandbox, attestation where justified |

## Construction example: consortium invoice exception model

Several construction partners want a shared model that predicts which invoices need manual review. Each company’s ERP contains commercially sensitive prices, vendors, projects, and approval history, so raw rows remain inside each company.

The coordinator publishes a signed feature contract and base model. Each participant creates features locally using point-in-time rules, trains in a restricted runtime, clips its update, and joins secure aggregation. A company with ten times more invoices cannot receive unlimited influence; the consortium evaluates both example-weighted and capped weighting.

One contractor mostly handles healthcare projects and another mostly handles road work, so the data is non-IID. The release suite includes per-company, project-type, vendor-novelty, and high-value-invoice metrics. The aggregate improves overall recall but increases false positives for the smallest contractor, so it is rejected pending personalization.

A malicious or broken participant submits an extreme update. Numeric bounds and clipping prevent overflow, and the candidate fails backdoor and cohort tests. The round is quarantined rather than released.

A healthcare-operations consortium can use the same architecture for operational forecasting, but patient-level privacy, institutional governance, data-use agreements, small-cohort risk, protected evaluation, and clinical boundaries must be explicit. Federated learning does not make a diagnostic use safe or authorized.

## Interview prompt

**Design a cross-silo federated-learning platform for construction companies that want a shared invoice-risk model without pooling raw ERP records. Participants differ greatly in size and data distribution, some disconnect, one may be malicious, and every production model must be auditable and reversible.**

## Worked answer

1. **Clarify objective and threat model.** Define prediction target, privacy unit, participants, legal boundaries, coordinator trust, collusion assumptions, model size, round cadence, and release owner.
2. **Choose the architecture.** Use cross-silo federation with a versioned coordinator, signed declarative training plans, restricted local runtimes, secure aggregation, registry, evaluation service, and staged deployment.
3. **Define contracts.** Version features, labels, time windows, schemas, preprocessing, base model, optimizer, clipping, weighting, privacy, cohort, and abort rules.
4. **Enroll safely.** Authenticate organization and workload, verify plan/runtime compatibility, enforce region and purpose, and issue a round-bound identity.
5. **Handle heterogeneity.** Track participation and distributions, compare weighting strategies, cap influence, consider personalized components, and evaluate worst cohorts.
6. **Protect contributions.** Validate numeric shape, clip updates, use secure aggregation under explicit threshold assumptions, add DP only with a stated unit and accountant, and never silently weaken privacy.
7. **Survive failure.** Make receipts idempotent, bind updates to round and base hash, checkpoint transfers, support protocol-safe dropout, and abort when quorum or privacy assumptions fail.
8. **Defend the model.** Test poisoning, backdoors, targeted cohorts, missing participants, and robust aggregation under realistic non-IID data.
9. **Govern release.** Register full lineage and privacy state, require independent evaluation and approval, then shadow, canary, stage, monitor, and retain rollback.
10. **Manage lifecycle.** Stop future participation on revocation, document historical-influence policy, expire credentials, rotate protocols, and preserve auditable round evidence without centralizing raw data.

**One-line conclusion:** Federated learning is safe only when locality, privacy, contribution integrity, non-IID evaluation, and release governance are explicit protocol properties—not assumptions inferred from the absence of a central dataset.

## References

- [Communication-Efficient Learning of Deep Networks from Decentralized Data](https://proceedings.mlr.press/v54/mcmahan17a.html)
- [Practical Secure Aggregation for Privacy-Preserving Machine Learning](https://dl.acm.org/doi/10.1145/3133956.3133982)
- [Advances and Open Problems in Federated Learning](https://www.nowpublishers.com/article/Details/MAL-083)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [TensorFlow Federated documentation](https://www.tensorflow.org/federated)
- [OpenFL documentation](https://openfl.readthedocs.io/)
