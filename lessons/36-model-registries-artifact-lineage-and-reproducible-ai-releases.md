# Model registries, artifact lineage, and reproducible AI releases

An AI system’s behavior is rarely determined by one model file. It also depends on code, prompts, policies, tools, data transformations, retrieval indexes, serving runtime, and deployment configuration.

A registry and lineage system makes that complete configuration identifiable, reviewable, promotable, reproducible, and revocable.

## Mental model

Treat an AI release as a **signed bill of materials plus evidence**, not “the latest model.”

A production release bundle points to immutable identities for:

- model weights or provider model version;
- tokenizer and inference runtime;
- application and workflow code;
- container image and software bill of materials;
- feature, parser, OCR, and embedding code;
- prompt and context-compiler templates;
- policy and tenant-configuration schemas;
- tool schemas and connector versions;
- training, tuning, and evaluation datasets;
- retrieval index generation and source snapshot;
- evaluation results, thresholds, and approvals;
- deployment configuration and infrastructure constraints.

The registry is the control plane for lifecycle and policy. Artifact stores hold bytes. Metadata catalogs describe datasets and business meaning. Deployment systems run approved bundles. Do not collapse these responsibilities into a mutable folder called “production.”

The key invariant is:

> Every production outcome can identify the exact release bundle and evidence that authorized it.

## When to use it

Use a registry and lineage system when:

- models, prompts, policies, or retrieval components change independently;
- the same artifact moves across development, staging, and production;
- regulated, contractual, or safety review is required;
- multiple tenants, regions, adapters, or model providers exist;
- rollback, incident investigation, or audit must be reliable;
- training and evaluation data evolve;
- an external model API may change aliases or retirement dates;
- teams need to reproduce a result months later.

A tiny prototype can begin with a versioned manifest in Git and immutable object storage. Introduce a dedicated registry when approval workflows, search, retention, access control, or multiple artifact types outgrow that approach.

## Registry, artifact store, and catalog

| Component | Owns | Must not become |
| --- | --- | --- |
| Registry | Versions, lifecycle, approvals, aliases, lineage references | Mutable artifact bucket |
| Artifact store | Immutable binaries, datasets, reports, manifests | Source of deployment policy |
| Metadata catalog | Dataset meaning, ownership, sensitivity, quality | Model promotion authority |
| Git repository | Reviewable code, templates, schemas, infrastructure | Storage for large secrets or mutable data |
| Deployment controller | Desired and observed runtime state | Place where release identity is invented |
| Evaluation service | Test execution and evidence | Sole approver for business risk |

Use content digests, not filenames, as identities. Human-friendly names and aliases may resolve to digests, but an alias such as `production` or `latest` must never be the only recorded identity.

## Release manifest

A release manifest can contain:

```yaml
release_id: invoice-assistant/2026-10-29.1
manifest_digest: sha256:...
model:
  provider: approved-provider
  model_id: model-family
  immutable_version: model-version-or-weight-digest
  tokenizer_digest: sha256:...
application:
  git_commit: 4a8...
  image_digest: sha256:...
prompt_bundle:
  digest: sha256:...
policy_bundle:
  digest: sha256:...
retrieval:
  embedding_model_digest: sha256:...
  index_generation: project-index-1842
  source_snapshot: 2026-10-29T11:00:00Z
tools:
  schema_bundle_digest: sha256:...
evaluation:
  suite_version: eval-suite/17
  dataset_snapshot: sha256:...
  report_digest: sha256:...
approvals:
  - role: model-risk-owner
    subject_digest: sha256:...
deployment:
  region: us-east
  tenant_class: construction-standard
```

The manifest itself is immutable. A correction creates a new version. Sign or attest the manifest and important artifacts, verify them during promotion and deployment, and record who or what performed each step.

Never place secrets in the manifest. Reference approved secret identities; inject values at runtime through the secret-management system.

## Lifecycle and state machine

Use explicit states such as:

1. **draft** — assembled but incomplete;
2. **candidate** — immutable and ready for validation;
3. **validated** — automated evidence passes;
4. **approved** — accountable reviewers accept defined risk;
5. **promoted** — eligible for a named environment or tenant class;
6. **deployed** — observed running with a deployment record;
7. **retired** — no new deployments;
8. **revoked** — blocked because trust or safety is invalid.

Promotion changes metadata pointing to an immutable bundle; it does not rebuild the bundle. Rebuilding between staging and production defeats “tested once, promoted unchanged.”

Require separation of duties for high-risk changes. The author of a model or policy should not be able to silently approve and deploy it alone.

## Artifact lineage

Represent lineage as a directed acyclic graph where nodes are immutable artifacts or executions and edges state relationships:

- dataset snapshot **used by** training run;
- code commit and configuration **produced** model;
- model and prompt bundle **evaluated by** suite;
- report and approval **authorized** release;
- release **deployed as** runtime instance;
- runtime **produced** prediction or tool proposal.

Record both data lineage and control lineage. A prediction needs the inputs and artifacts that shaped it; a deployment needs the evaluations and approvals that permitted it.

Each execution record should include:

- stable run or operation ID;
- actor/workload identity;
- input and output artifact digests;
- code, environment, and configuration identity;
- start/end time and status;
- hardware/runtime details when material;
- parent execution;
- policy decision and approval references;
- logs and metrics subject to minimization and retention.

Avoid free-text lineage that cannot be queried. Use typed edges and validate required ancestry before promotion.

## Dataset and index lineage

“Dataset v3” is insufficient. Snapshot identity should describe:

- source systems and extraction boundaries;
- query or selection logic;
- valid-time and ingestion-time cutoff;
- tenant and authorization scope;
- schema and feature definitions;
- labels and adjudication process;
- consent, retention, license, and permitted purpose;
- redaction/de-identification version;
- exclusions and known gaps;
- content digest or immutable snapshot ID.

For retrieval systems, record parser, chunker, OCR, embedding model, metadata schema, authorization-filter version, source revisions, tombstones, and index generation.

An index can be regenerated, but only if its exact source set and transformation chain are known. A deleted or corrected source must propagate to every derived index and cached summary.

## Reproducibility versus repeatability

Exact bit-for-bit reproduction may be impossible when:

- GPU kernels are nondeterministic;
- distributed training order changes;
- external APIs hide weights or change infrastructure;
- sampled generation uses randomness;
- upstream data is mutable;
- hardware and libraries differ.

Define the required level:

- **artifact reproducibility:** retrieve the exact original bytes;
- **environment reproducibility:** reconstruct code, dependencies, hardware class, and configuration;
- **statistical reproducibility:** reproduce quality within declared tolerances;
- **decision reproducibility:** reconstruct why a release or action was permitted;
- **behavioral reproducibility:** pass the same versioned evaluation contract.

Pin dependencies, images, seeds, datasets, and provider versions where possible. When exact replay is impossible, record the limitation and preserve enough evidence for statistical and decision-level reconstruction.

## Build and supply-chain integrity

A secure build path:

1. checks out an approved commit;
2. resolves locked dependencies;
3. builds in an isolated runner;
4. scans source, dependencies, containers, and secrets;
5. generates an SBOM and provenance attestation;
6. stores artifacts by digest in immutable storage;
7. signs or attests the release subject;
8. runs evaluation against versioned data;
9. records approvals against the exact digest;
10. promotes the same digest;
11. verifies signatures, policy, and revocation status at deployment.

Protect the registry itself with least privilege, strong workload identity, audit logs, backups, and change monitoring. If attackers can rewrite lineage or move the production alias, artifact signing alone is insufficient.

Third-party model and container artifacts need origin, license, vulnerability, model-card, and allowed-use review. Mirror critical artifacts when contracts permit, because an upstream deletion can prevent recovery.

## Evaluation and promotion gates

A gate should specify:

- release type and risk tier;
- required suites and slices;
- minimum quality and maximum regression;
- latency, throughput, and cost budgets;
- security and prompt-injection tests;
- privacy, residency, and licensing checks;
- schema and backward-compatibility checks;
- human-review requirements;
- expiry period for evidence;
- permitted exceptions and accountable approver.

Do not gate only on average accuracy. For invoice extraction, evaluate totals, tax, vendor, line items, currency, handwriting/OCR quality, language, missing pages, duplicate invoices, and adversarial documents. For tool use, evaluate argument correctness, authorization, abstention, stale approval, and outcome-unknown handling.

An exception is a versioned, time-bounded artifact with scope, rationale, compensating controls, and owner—not a message in chat.

## Deployment, rollout, and rollback

Resolve a promoted release to immutable digests before rollout. Deploy by digest and report observed identity from every instance.

Use canary, shadow, or cohort rollout based on risk. Compare:

- task success and critical-slice quality;
- schema validity and citations;
- tool proposals and side effects;
- latency, errors, saturation, and cost;
- tenant fairness;
- operator overrides and complaints.

Rollback must consider compatibility. An older application may not understand a newer workflow state, database schema, prompt contract, or index. Prefer forward-compatible schemas, expand/contract migrations, versioned APIs, and the ability to disable a risky capability independently.

Keep rollback artifacts and dependencies available for the declared recovery window. If a release caused bad derived data or external actions, rolling back compute does not undo them; launch repair, compensation, or reconciliation workflows.

## External model providers

For managed models:

- pin an immutable provider version when available;
- record endpoint, region, deployment, API version, and safety configuration;
- treat aliases as mutable routing metadata;
- monitor retirement and behavior-change notices;
- keep task-specific evaluation baselines;
- test approved fallbacks before incidents;
- record the actual provider response identity when exposed;
- prohibit routing sensitive data to an unapproved deployment.

If the provider offers no stable version, acknowledge weaker reproducibility. Increase continuous evaluation, archive representative inputs and permitted outputs, use change detection, and define a migration plan.

## Construction operations example

A construction platform upgrades invoice extraction and change-order matching.

The release bundle includes:

- OCR engine and configuration;
- invoice parser code;
- extraction model and tokenizer;
- prompt and output schema;
- vendor normalization table;
- cost-code mapping version;
- retrieval index generation for contracts and change orders;
- approval policy bundle;
- ERP connector and tool schema;
- evaluation dataset with low-quality scans and disputed changes;
- container digest, SBOM, evaluation report, and approvals.

The candidate improves aggregate field accuracy but regresses handwritten tax values and duplicates an ERP proposal when a timeout occurs. The gate blocks promotion despite the average gain.

After fixes, the system shadows production documents without writing to ERP, then canaries read-only extraction for selected projects. Tool execution remains on the old release until idempotency and reconciliation tests pass.

Every extracted invoice records the release ID and source-document digest. If the vendor mapping is later corrected, the team can identify affected outputs, reprocess them with a new release, and route consequential differences for review.

## Healthcare operations variation

A denied-claim assistant’s bundle also records:

- de-identification and PHI-handling versions;
- FHIR/HL7 mapping and terminology releases;
- cohort and label definitions;
- purpose-of-use and consent policy;
- approved model region and business-associate constraints;
- fairness and clinical-safety slices;
- human-review protocol;
- retention and deletion obligations.

Training data, evaluation data, and production inputs remain purpose-scoped. A registry entry does not grant access to PHI; authorization is enforced by the underlying data systems.

Promotion requires privacy, security, clinical/operations, and platform evidence appropriate to risk. If a terminology release or payer rule changes, lineage identifies affected evaluations, indexes, outputs, and deployments. Consequential claim or scheduling actions continue to require current authorization and audited approval.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| One bundle per release | Clear reproducibility | More metadata discipline |
| Independent component promotion | Faster changes | Combinatorial compatibility |
| Immutable artifacts | Audit and rollback | Storage and retention cost |
| Mutable aliases | Operational convenience | Hidden drift if identity not recorded |
| Dedicated registry | Search, policy, lifecycle | Platform complexity |
| Git manifest only | Simple and reviewable | Weak at large artifacts and rich queries |
| Strict pinning | Repeatability | Slower security updates |
| Floating dependencies | Fast patch uptake | Non-reproducible builds |
| Broad evaluation suite | Better risk coverage | Time and compute cost |
| Risk-tiered gates | Efficient governance | Requires sound classification |
| Full lineage retention | Strong investigations | Privacy and storage burden |
| Managed model API | Less infrastructure | Weaker artifact control |
| Automatic promotion | Speed | Unsafe for high-risk changes |
| Human approval | Accountability | Queue delay and reviewer load |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Deploy `latest` | Runtime identity drifts | Resolve and deploy digest |
| Registry stores mutable files | Evidence can be rewritten | Immutable content-addressed store |
| Rebuild for production | Untested artifact ships | Promote same digest |
| Model version excludes prompt | Behavior changes invisibly | Whole-system release manifest |
| Dataset name lacks snapshot | Training cannot be reconstructed | Immutable selection and cutoff |
| Index lacks source lineage | Stale/deleted evidence persists | Generation and tombstone lineage |
| Gate uses aggregate score | Critical slice regresses | Risk-sliced thresholds |
| Approval references alias | Subject changes after approval | Bind approval to digest |
| External API alias changes | Silent behavior shift | Pin version or continuous change tests |
| Registry has broad write access | Supply-chain compromise | Least privilege, audit, signing |
| Signature checked without policy | Untrusted signer accepted | Verify identity, subject, and policy |
| Rollback ignores schema | Older service corrupts state | Compatibility contract and migration plan |
| Rollback assumed to undo actions | External effects remain | Reconciliation/compensation |
| Secrets embedded in manifest | Credential exposure | Runtime secret references |
| Lineage logs raw PHI | Privacy incident | Minimized metadata and controlled evidence |
| Deleted source survives cache/index | Retention violation | Propagated deletion and verification |
| Evidence never expires | Stale approval promotes change | Gate freshness and revalidation |
| Nondeterminism is hidden | False reproduction claim | Declare required reproducibility level |

## What to measure

Track unregistered artifacts; releases with complete manifests; digest/signature verification; lineage completeness; build reproducibility; evaluation freshness; gate pass/fail and exception rate; time from candidate to decision; approval queue age; deployment identity mismatch; canary regressions; rollback success and time; artifact vulnerability age; provider-version drift; dataset/index deletion propagation; affected-output discovery time; and percentage of production outcomes traceable to an approved release.

The most important operational test is not “can we find the model?” It is “can we explain, reproduce within declared limits, stop, and repair the exact system that produced this outcome?”

## Interview prompt

> Design a registry, lineage, and release system for a multi-tenant construction AI platform. It uses managed and self-hosted models, prompts, retrieval indexes, policies, and ERP tools. Teams need safe promotion, rollback, audit, and reproducibility. Adapt the design for healthcare operations with PHI.

Cover artifact identity, manifests, lifecycle, lineage, dataset and index snapshots, build integrity, evaluation gates, approvals, deployment, provider models, rollback, deletion, multi-tenancy, and failure recovery.

## Worked answer

**One-line conclusion:** Promote an immutable, signed whole-system release bundle by digest, require fresh risk-based evidence and approvals, and trace every deployment and outcome back through reproducible artifact and data lineage.

“I would separate immutable artifact storage from a metadata registry, data catalog, evaluation service, and deployment controller. Every model, image, prompt, policy, tool schema, dataset, and index generation has a digest. A release manifest references those digests plus code, runtime, configuration, evaluation, and approvals.

Builds run in isolated CI from an approved commit and locked dependencies. They emit an image, SBOM, provenance, signatures, and evaluation report. Promotion changes an environment alias to an already-tested digest; production never rebuilds or deploys `latest`. The deployer verifies signer identity, policy, approval subject, evidence freshness, and revocation status.

The lineage graph connects source snapshots and transformations to models and indexes, evaluation to releases, releases to deployments, and deployments to outcomes. Dataset snapshots include selection logic, time cutoff, schema, labels, permissions, and deletion obligations. Managed-model endpoints record provider, immutable version where available, region, API version, and observed identity; mutable aliases receive continuous evaluation and change detection.

Gates are risk-tiered and slice-specific. Construction invoice releases must pass poor-scan, handwritten, currency, duplicate, authorization, idempotency, and uncertain-ERP tests before tool execution can canary. Healthcare adds PHI eligibility, FHIR/terminology lineage, fairness, privacy, and accountable clinical or operational review.

Rollback resolves a known compatible digest and does not pretend to undo data or external effects. Schema compatibility, repair jobs, and reconciliation are planned separately. I would monitor lineage completeness, signature verification, stale evidence, runtime identity mismatch, canary quality, exception use, rollback recovery, and the time required to find every outcome affected by a bad artifact.”

## Practice lab

Design the release path for an invoice assistant where:

- a new OCR model improves clean scans but regresses handwriting;
- the prompt and ERP schema change together;
- the embedding index is rebuilt from corrected contracts;
- one dependency has a critical vulnerability;
- the managed-model alias changes behavior;
- an approval is attached to an alias rather than a digest;
- the canary writes one duplicate ERP proposal;
- rollback code cannot read the newest workflow state;
- one source contract must be deleted from all derived artifacts.

Specify the manifest, lineage edges, gate evidence, signer and approvers, rollout cohorts, runtime verification, rollback/repair plan, and affected-output query.

Then adapt the controls for PHI, FHIR terminology changes, and a healthcare model hosted only in an approved region.

## References

- [MLflow Model Registry documentation](https://mlflow.org/docs/latest/ml/model-registry/)
- [OCI Image and Distribution Specifications](https://github.com/opencontainers/image-spec)
- [SLSA supply-chain security framework](https://slsa.dev/spec/)
- [Sigstore documentation](https://docs.sigstore.dev/)
- [NIST Secure Software Development Framework, SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final)
- [OpenLineage specification](https://openlineage.io/docs/spec/)
- [W3C PROV overview](https://www.w3.org/TR/prov-overview/)
