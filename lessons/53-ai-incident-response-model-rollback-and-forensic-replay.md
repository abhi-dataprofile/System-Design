# AI incident response, model rollback, and forensic replay

A production AI incident is not limited to model downtime. The system may remain available while producing unsafe recommendations, leaking tenant data, following poisoned context, repeating tool actions, or silently changing after a provider update.

Incident response must contain harm, preserve trustworthy evidence, reconcile external side effects, restore a known-good system, and convert the event into durable controls.

## Mental model: contain the decision path, not just the model

Treat AI behavior as a versioned decision path:

```text
input
 -> parser / retrieval / memory
 -> prompt and policy
 -> model and runtime
 -> tools and human approval
 -> external side effect
 -> delayed outcome
```

An incident can originate at any node. Rolling back model weights does not remove a poisoned retrieval index, malicious memory, changed provider alias, compromised tool credential, or already executed ERP write.

Use six phases:

1. **Detect and declare:** identify symptoms, impact, scope, and incident owner.
2. **Contain:** stop or narrow harmful paths without destroying evidence.
3. **Preserve:** freeze versioned artifacts, logs, approvals, and external outcomes.
4. **Eradicate:** remove the cause and invalidate contaminated state.
5. **Recover:** restore through a tested, monitored path.
6. **Learn:** repair control gaps and add permanent regression tests.

Keep two truths separate:

- **system truth:** what the AI pipeline recorded or intended;
- **external truth:** what authoritative systems and people actually received or changed.

## When to use an AI-specific response plan

Extend normal security and reliability response when the system uses:

- probabilistic models whose failures may be semantic rather than HTTP errors;
- prompts, retrieval, memory, tools, or agents that change behavior independently;
- hosted model aliases that providers can update;
- human approval or external side effects;
- sensitive, regulated, or multi-tenant data;
- delayed labels that reveal harm after deployment.

Do not create a separate incident organization for AI. Integrate with existing security, privacy, reliability, legal, and business-continuity processes, while adding AI-specific evidence and containment controls.

## Detection and severity

Signals should cover more than latency:

- policy violations and cross-tenant retrieval;
- groundedness, abstention, and citation regressions;
- unusual tool-call rate, scope, or payload;
- output-distribution or refusal changes;
- reviewer corrections and escalation rates;
- provider, prompt, model, index, or policy version changes;
- secret, PHI, or regulated-data detectors;
- duplicate or outcome-unknown external actions;
- cost and token velocity;
- canary-versus-baseline divergence;
- credible user or operator reports.

Severity should combine harm, exposure, reversibility, spread, and evidence confidence. A single confirmed cross-tenant disclosure may be more severe than thousands of harmless malformed outputs.

Declare uncertainty explicitly. “Scope unknown” is not “no impact.”

## Incident command and decision log

Assign clear roles:

| Role | Responsibility |
|---|---|
| Incident commander | Owns priorities, status, and decisions |
| Technical lead | Investigates and coordinates containment/recovery |
| AI/ML lead | Examines models, prompts, evals, drift, and routing |
| Security/privacy lead | Handles access, disclosure, evidence, and notification |
| Operations/domain lead | Assesses business and user impact |
| Communications lead | Maintains accurate internal and external updates |
| Scribe | Records timeline, hypotheses, actions, approvals, and evidence IDs |

Maintain an append-only decision log. Record who changed which control, when, why, expected effect, actual effect, and rollback condition. Avoid conducting the incident only in chat messages.

## Containment ladder

Predefine controls from narrowest to broadest:

1. disable one prompt, memory source, retriever, tenant, tool, or provider route;
2. force read-only or draft-only behavior;
3. require human review for affected decisions;
4. route eligible traffic to a known-good evaluated release;
5. block a data class, region, or workflow;
6. disable AI while preserving a manual fallback;
7. shut down the service if no bounded path remains.

Containment must be policy-aware. Do not fail over PHI to an unapproved region, replace an unsafe model with an unevaluated one, or queue side effects that will fire later without renewed authorization.

Kill switches should be independently enforceable, audited, tested, scoped, and available even when the AI control plane is degraded.

## Preserve evidence before changing the system

Capture immutable identifiers for:

- request, trace, session, tenant, user, device, and workload;
- input and attachment hashes;
- parser, OCR, chunker, embedding, index snapshot, and retrieved record versions;
- prompt template, assembled context, policies, safety filters, and memory;
- model weights or provider version, tokenizer, runtime, route, and generation settings;
- tool schemas, arguments, credentials or grant IDs, approvals, idempotency keys, and outcomes;
- human review and user-visible output;
- deployment, feature flag, and configuration history;
- relevant metrics, logs, queues, caches, and audit records.

Preserve enough to explain behavior, but do not copy secrets or sensitive content into an unrestricted incident folder. Use legal hold, access control, encryption, retention limits, and evidence manifests. Record collection method and hashes so evidence provenance is defensible.

## Forensic replay

Replay asks: “Given the same authorized evidence and release, can we reproduce the decision path?”

Use an isolated environment with no production side effects. Stub tools or route them to deterministic simulators. Pin every available artifact and record sources of nondeterminism.

Classify replay:

- **Exact replay:** identical immutable artifacts and deterministic settings.
- **Bounded replay:** same release but probabilistic output; compare distributions and invariants.
- **Counterfactual replay:** change one component to test a hypothesis.
- **Current-state reenactment:** useful diagnostically, but not evidence of past behavior.

Never replay production tool calls against live systems. Redact or synthesize sensitive inputs when the question does not require originals. A hosted model may no longer be available; record that limitation rather than pretending a current alias is equivalent.

## Rollback is a dependency graph

A release includes more than a model:

- model or provider version;
- runtime and tokenizer;
- preprocessing and feature code;
- prompt and policy;
- retrieval corpus, embedding model, and index;
- tool schemas and permissions;
- memory state;
- feature flags and routing;
- application code and configuration.

Rollback the smallest complete, compatible bundle. Check schema and index compatibility, data migrations, credential scopes, cache contents, and queued work.

Some state must be invalidated rather than rolled back: poisoned memory, contaminated embeddings, leaked credentials, or generated drafts derived from bad evidence. Preserve the old state for investigation, quarantine it from production, and rebuild trusted derivatives from clean sources.

## External side effects and reconciliation

AI rollback cannot reverse an email, payment, schedule change, medication transport request, or ERP update.

For every affected intent, classify the outcome:

- definitely not executed;
- definitely executed;
- rejected;
- partially executed;
- outcome unknown.

Query the authoritative system using stable intent and idempotency IDs. Compensate only through approved business workflows. Do not blindly retry unknown outcomes or delete audit history.

Notify impacted operators with specific objects and actions to review. A broad “the model was wrong” notice is less useful than a list of affected invoices, RFIs, rooms, or work orders with evidence and current status.

## Recovery strategy

Recover progressively:

1. prove the containment boundary holds;
2. validate a known-good bundle against incident regressions and representative evaluations;
3. restore read-only or shadow traffic;
4. enable a small canary cohort;
5. compare safety, quality, tool behavior, latency, and cost;
6. expand by tenant, region, and workload;
7. keep enhanced monitoring and an immediate rollback path;
8. close only after side effects, queues, contaminated state, and notifications are reconciled.

Do not use aggregate quality alone. The incident class may hide inside a rare tenant, language, document type, or tool path.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Broad shutdown | Fast harm containment | Operational disruption and loss of useful safe functions |
| Narrow feature kill | Preserves service | Incorrect scope can allow continued harm |
| Full-fidelity logging | Strong investigation and replay | Privacy, security, retention, and cost exposure |
| Data-minimized evidence | Lower exposure | May limit root-cause analysis |
| Automatic rollback | Fast recovery for known signals | Can restore an incompatible or also-vulnerable release |
| Manual rollback approval | Better contextual judgment | Slower containment |
| Retaining prior indexes/models | Reproducibility and rapid rollback | Storage, licensing, and vulnerability risk |
| Provider failover | Restores capacity | Semantic drift, policy eligibility, and new failure modes |
| Immediate customer notice | Rapid protective action | Premature or inaccurate scope |
| Delayed confirmed notice | More accurate facts | Prolonged exposure or missed obligations |

## Failure modes and defenses

| Failure mode | Defense |
|---|---|
| Team rolls back only model weights | Release manifest covering prompt, policy, retrieval, tools, runtime, and config |
| Logs omit retrieved evidence or version | Trace-level lineage with immutable artifact IDs |
| Investigation mutates evidence | Read-only snapshots, manifests, hashes, and access audit |
| Replay repeats a real side effect | Isolated replay environment and deterministic tool stubs |
| Failover violates residency or safety | Hard eligibility filters before recovery routing |
| Poisoned memory survives rollback | Quarantine and rebuild derived state from trusted sources |
| Provider alias changed without notice | Pin versions where possible and continuously fingerprint behavior |
| Queue later executes contained actions | Freeze queues; reauthorize and review before release |
| Unknown ERP outcome is retried | Reconcile by intent/idempotency ID against authoritative state |
| Aggregate metrics hide one harmed cohort | Slice by tenant, region, language, workflow, and input class |
| Sensitive incident data spreads through chat | Restricted case workspace with references instead of copied content |
| Postmortem blames one operator | Focus on control and system failures; assign measurable remediation owners |
| Fix ships without regression coverage | Convert incident artifacts into sanitized permanent tests |

## Construction example: poisoned drawing causes unsafe RFI drafts

A construction assistant ingests a subcontractor PDF containing hidden instructions. The retrieval system begins inserting those instructions into drawing questions, and an agent drafts RFIs that cite the wrong revision. One draft may have been submitted to the project system.

The team disables the affected corpus and RFI submission tool, preserving read-only drawing access from a known-good index. It freezes the agent queue so contained drafts cannot submit later. Evidence includes the PDF hash, parser version, chunks, embedding model, index snapshot, retrieved passages, assembled prompt, model route, tool payload, approval record, and idempotency key.

The response team replays the trace with a tool simulator. Rolling back the language model alone reproduces the failure because the poisoned chunk remains. They quarantine the corpus, rebuild the index from trusted documents, invalidate derived memory, rotate the ingestion credential, and add an indirect-prompt-injection regression.

The project system is queried for the submission intent. If the RFI exists, a project manager follows the normal correction workflow; if outcome is unknown, the system does not retry. Recovery starts read-only, then draft-only, then canary submission after evaluation and human approval.

A healthcare-operations variant might involve poisoned transport instructions or cross-patient retrieval. Containment must preserve essential manual operations, minimize PHI in evidence, reconcile every external task, and involve privacy and clinical-operations leadership.

## Interview prompt

**Design incident response for a multi-tenant AI operations assistant after reports that one customer’s documents influenced another customer’s answers and an agent may have submitted incorrect ERP actions. Explain containment, evidence, replay, rollback, recovery, and side-effect reconciliation.**

## Worked answer

1. **Declare and organize.** Assign incident command, technical, AI, security/privacy, domain, communications, and scribe roles. State that scope is unknown.
2. **Protect users first.** Disable the suspected retrieval path and side-effecting tools, freeze related queues, preserve safe read-only/manual workflows, and block unsafe failover.
3. **Scope by lineage.** Search traces by corpus, index snapshot, model route, tenant, prompt, tool, and time window. Identify exposed outputs and attempted actions.
4. **Preserve evidence.** Snapshot manifests, hashes, retrieval results, prompts, policies, model/provider versions, tool payloads, approvals, deployments, and authoritative outcomes under restricted access.
5. **Replay safely.** Use isolated infrastructure and tool stubs. Reproduce the path, then change one component at a time to locate the cause.
6. **Rollback completely.** Restore a compatible known-good release bundle; quarantine poisoned source and derived indexes, caches, and memory rather than merely reverting weights.
7. **Reconcile effects.** Classify every intent as executed, not executed, rejected, partial, or unknown. Query authoritative systems and use approved correction workflows; never retry blindly.
8. **Validate recovery.** Run sanitized incident regressions and broad evaluations, then progress through shadow, read-only, draft-only, canary, and staged expansion.
9. **Communicate precisely.** Provide affected tenants and operators with confirmed scope, impacted objects, protective actions, and correction status, following notification obligations.
10. **Prevent recurrence.** Add ingestion trust controls, tenant-bound retrieval invariants, queue reauthorization, complete release lineage, kill-switch drills, and owned remediation deadlines.

**One-line conclusion:** Effective AI incident response contains the entire decision path, preserves version-complete evidence, reconciles real-world effects, and restores service only through a tested known-good bundle.

## References

- [NIST SP 800-61 Rev. 3 — Incident Response Recommendations and Considerations for Cybersecurity Risk Management](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [NIST AI RMF Generative AI Profile](https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence)
- [MITRE ATLAS](https://atlas.mitre.org/)
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)
- [OpenTelemetry Trace semantic conventions](https://opentelemetry.io/docs/specs/semconv/general/trace/)
- [SLSA provenance](https://slsa.dev/spec/v1.0/provenance)
