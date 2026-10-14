# 21 — Privacy engineering, retention, and deletion workflows

Previous: [Human review systems, approval queues, and operator experience](20-human-review-systems-approval-queues-and-operator-experience.md). Study time: 50–60 minutes including the exercise.

## Mental model

Privacy is a data-lifecycle property, not a consent banner or an encryption checkbox. For every sensitive datum, the system must be able to answer:

```text
why collected → where stored → who may use it → what derived from it
              → how long retained → how access/deletion propagates → what proves completion
```

Treat deletion as a distributed workflow across authoritative stores, replicas, indexes, object versions, caches, analytics, logs, queues, backups, external processors, and AI artifacts. A row disappearing from the primary database is only the beginning.

> Collect the minimum data for an explicit purpose, enforce that purpose at use time, and make retention and deletion observable workflows with documented exceptions.

This lesson explains engineering patterns, not legal advice. Actual obligations depend on contracts, jurisdiction, record type, healthcare role, litigation holds, and organizational policy.

## When to use these controls

Apply privacy lifecycle controls whenever a system handles:

- personal, employee, customer, patient, financial, location, biometric, or behavioral data;
- confidential project drawings, access codes, bids, or subcontractor records;
- prompts, model outputs, retrieval traces, reviewer notes, or support transcripts;
- uploaded files whose contents are not known before ingestion;
- data copied into analytics, evaluation, training, or third-party services;
- records subject to contractual retention, legal hold, access, correction, export, or deletion requests.

Do not wait until a deletion request arrives. Retrofitting lineage into an undocumented data estate is expensive and unreliable.

## Start with a data inventory

Create a machine-readable catalog at dataset and field level:

| Property | Question |
| --- | --- |
| Owner | Who approves collection, access, and lifecycle changes? |
| Subject/resource | Person, project, patient, vendor, employee, document? |
| Classification | Public, internal, confidential, regulated, highly sensitive? |
| Purpose | Which business outcome justifies processing? |
| Authority | Consent, contract, operational need, legal requirement? |
| Location | Database, object store, index, cache, log, SaaS, device? |
| Lineage | Which sources and derived products feed this copy? |
| Access | Which roles, services, tenants, and support workflows? |
| Residency | Which region and which cross-border transfers? |
| Retention | Trigger, duration, disposal action, exception? |
| Recovery | Which replicas, snapshots, and backups contain it? |
| Processor | Which external provider receives it and under what contract? |

Inventory data flows, not only databases. An invoice may be copied into OCR output, chunks, embeddings, prompts, traces, evaluation cases, reviewer screenshots, warehouse tables, and support tickets.

Use stable data-subject and resource identifiers so the system can locate related records without relying on names or free-text search alone.

## Data minimization and purpose limitation

Minimization has four dimensions:

1. **fields:** collect only attributes required for the purpose;
2. **scope:** restrict rows, tenants, projects, patients, and time ranges;
3. **precision:** use age band instead of birth date or region instead of exact location when sufficient;
4. **duration:** retain only as long as the approved purpose requires.

Enforce purpose at access time. A user authorized to view an invoice for payment review is not automatically authorized to reuse its data for model training or marketing analytics.

Useful patterns:

- purpose-scoped service identities and access tokens;
- server-derived tenant, project, and patient context;
- views that omit unnecessary fields;
- tokenization or pseudonymous IDs for analytics;
- aggregate metrics with minimum group sizes;
- separate production, evaluation, and training permissions;
- outbound allowlists and sensitive-field filters;
- just-in-time privileged access with audit and expiry.

Pseudonymization reduces exposure but is not anonymization when a lookup table, auxiliary data, or rare attributes can reconnect a record to a person.

## Retention policy as executable data

A retention schedule should specify:

```text
record class + purpose + start trigger + duration + action + exceptions + owner
```

Examples of start triggers include project closeout, invoice settlement, contract termination, employee separation, patient encounter, consent withdrawal, or superseding document version. “Created at” is not always the right clock.

Store a computed `retention_due_at`, policy version, and lifecycle state with the resource. Recompute safely when policies change, but never shorten legally required retention or override a hold without authorization.

Typical lifecycle states:

```text
ACTIVE → ARCHIVED → DELETION_ELIGIBLE → DELETING → DELETED
                    ↘ HELD → RELEASED ↗
DELETING → PARTIAL_FAILURE → RETRYING
```

Archival is not deletion. Moving data to cheaper storage changes availability and cost, not privacy exposure.

## Legal and operational holds

A hold temporarily overrides normal disposal for a defined scope and reason. Model it as a first-class record containing:

- hold ID and authorized issuer;
- tenant, subject, project, record class, and time scope;
- reason category with restricted details;
- effective time and release criteria;
- immutable audit events;
- affected systems and last propagation status.

Apply holds before deletion eligibility is evaluated. If a hold arrives while deletion is running, stop unfinished destructive steps and record what was already removed. Never restore deleted data merely to make a later hold appear complete.

Keep hold details access-controlled. A generic “retained due to authorized hold” status may be safer than exposing investigation details.

## A deletion control plane

Use an orchestrated workflow rather than a best-effort cascade.

1. Authenticate the requester and authorize the request type.
2. Resolve subject, tenant, resource, and applicable jurisdiction/policy.
3. discover authoritative and derived records through the inventory and lineage graph.
4. Check retention requirements, active holds, fraud/security exceptions, and contractual constraints.
5. Freeze new derivations or mark the resource deletion-pending.
6. Write a deletion manifest with scoped targets and versions.
7. Delete or irreversibly detach online authoritative data.
8. Publish tombstones to downstream consumers.
9. Purge indexes, caches, objects, analytics, and AI artifacts.
10. Notify external processors and track acknowledgements.
11. Record backup treatment and prevent normal restoration from resurrecting data.
12. Verify targets, document exceptions, and close only when evidence is complete.

The workflow must be idempotent and resumable. Each target uses a stable deletion operation ID. Retries should repeat the same intent without widening scope.

## Tombstones and distributed propagation

Hard deletion without a tombstone can cause resurrection when an old event, replica, or connector republishes the record.

A tombstone should include:

- tenant and stable resource/subject ID;
- deletion operation ID;
- resource version or deletion epoch;
- effective time;
- policy class;
- minimal routing metadata;
- no copied sensitive payload.

Consumers apply tombstones monotonically: an older create/update event cannot overwrite a newer deletion epoch. Retain the minimal tombstone long enough to cover event retention, offline consumers, and replay windows.

Separate “not found” from “deleted” internally. Externally, authorization and enumeration risks may require identical responses.

## Databases, files, and replicas

For relational data:

- define ownership and foreign-key disposal behavior;
- avoid accidental cascade into records with separate retention duties;
- delete or anonymize free-text fields and JSON blobs;
- purge change-data-capture streams and derived tables under their schedules;
- ensure read replicas receive deletion before completion is claimed.

For object storage:

- delete all relevant versions, multipart uploads, thumbnails, OCR outputs, and derived files;
- account for object-lock and write-once policies;
- use lifecycle rules as a backstop, not the sole request workflow;
- record version IDs and deletion markers.

For caches and search:

- invalidate by stable resource and subject IDs;
- remove lexical documents, vector entries, metadata, autocomplete, and semantic caches;
- verify tenant namespaces;
- design cache TTL as the maximum residual exposure if invalidation fails.

## Logs, audit, and observability

Logs are often the hardest uncontrolled copy. Prevent the problem:

- never log full prompts, tokens, documents, credentials, or raw clinical/financial payloads by default;
- use structured allowlists rather than redaction after serialization;
- hash or tokenize correlation identifiers when direct identity is unnecessary;
- keep debug payload capture off by default, time-bounded, approved, and isolated;
- set retention per log class;
- keep production data out of test fixtures.

Audit records may need longer retention than business content. Store the minimal fact that an action occurred—actor, time, purpose, resource reference, decision, policy version—without duplicating the underlying sensitive payload. If the resource is deleted, preserve only what the approved audit policy requires.

Deletion telemetry should use operation IDs and counts, not the deleted values.

## Backups and recovery

Immediate removal from immutable backups may be impractical or may weaken disaster recovery. Define and document a backup exception:

- backups are encrypted, access-restricted, immutable, and expire on a bounded schedule;
- deleted data is not restored into normal service;
- every restore enters an isolated recovery workflow;
- current deletion tombstones are replayed before applications reopen;
- expired backup media is cryptographically or physically disposed;
- the deletion response distinguishes live-system completion from scheduled backup expiry.

Crypto-shredding can make data unreadable by destroying a unique encryption key. It works only if keys are sufficiently granular, all copies are encrypted under the affected key hierarchy, plaintext and derived copies are gone, and key backups follow the same control. A tenant-wide key is too coarse for deleting one document.

## AI-specific copies

AI systems create less obvious derivatives:

- parsed text, page images, captions, chunks, and embeddings;
- prompts, responses, tool arguments, traces, and conversation memory;
- evaluation examples, reviewer labels, synthetic variants, and fine-tuning data;
- model-provider retention and abuse-monitoring copies;
- semantic and response caches;
- checkpoints or adapters trained on customer data.

Every artifact needs source lineage: tenant, source resource/version, transformation version, storage location, and retention class.

Deleting an embedding removes retrievability from that index; it does not delete the source chunk or model trace. Conversely, deleting a source object does not remove its vector.

For training data, decide before training whether deletion from future datasets is sufficient or whether model-level remediation is promised. Removing a record from a trained model is technically different from deleting a database row and may require retraining, model unlearning with validated guarantees, or an explicit documented exception. Never promise precise erasure from model weights without a proven process.

Third-party model calls should use contractual zero-retention or approved retention settings where required, minimum necessary fields, regional routing, and a processor inventory.

## Data access, correction, and export

Subject requests are identity and authorization workflows. Protect against both wrongful disclosure and wrongful deletion.

- verify identity proportionate to risk;
- distinguish a person from an account, tenant, patient, employee, or project resource;
- prevent one tenant administrator from exporting another tenant’s data;
- search structured sources through stable identifiers;
- review free text carefully because it can mention multiple people;
- produce machine-readable exports with provenance and understandable descriptions;
- propagate corrections to derived indexes and future AI context;
- record deadlines, exceptions, decisions, and delivery evidence.

Do not expose security-sensitive details, other people’s data, privileged notes, or secrets in a raw database dump.

## Construction example

A subcontractor requests removal of personal contact details after a project closes, while the company must retain contractual invoice and safety records.

1. The request is identity-verified and mapped to the subcontractor’s stable person ID.
2. Policy separates personal profile/contact data from project records with independent retention.
3. An active claim hold preserves specified project correspondence and invoice evidence.
4. The workflow deletes an unused mobile number from the profile, directory cache, CRM sync, search documents, embeddings, and notification provider.
5. Required project records keep a pseudonymous party reference and minimal audit facts; access becomes restricted to the retained purpose.
6. A tombstone prevents an offline CRM export from repopulating the phone number.
7. Object versions, OCR text, analytics, support tickets, and AI traces are checked through lineage.
8. Backups age out under policy; any restore replays the tombstone before service.
9. The completion record lists deleted systems, retained categories with policy reasons, processor acknowledgements, failures, and next retry.

The system does not treat “delete this user” as permission to destroy financial or safety records that have a different authorized retention obligation.

## Healthcare operations variation

A patient asks for access and correction while an encounter record is subject to healthcare retention rules. The platform may be a covered entity, business associate, or neither; that role changes obligations and must be established with counsel and contracts.

The workflow verifies identity, finds records by patient and encounter identifiers, separates designated clinical records from operational telemetry, and routes corrections as amendments rather than silently rewriting clinical history. Derived search indexes and future AI context reflect the corrected status while provenance preserves the authorized amendment trail.

Minimum-necessary access applies to operators and AI tools. PHI is excluded from general logs and model training unless explicitly authorized. If deletion is not permitted for a retained clinical record, the system communicates the applicable reason, restricts further use to the allowed purpose, deletes unnecessary derivatives, and schedules disposal when the policy allows.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Collect more data | Future flexibility | Larger breach and governance surface |
| Strict minimization | Lower exposure | Less debugging and analytics context |
| Hard delete | Strong live-store removal | Broken references and audit tension |
| Pseudonymize | Useful analytics | Re-identification remains possible |
| Central deletion orchestrator | Visibility and retries | Control-plane dependency |
| Event tombstones | Prevent resurrection | Minimal metadata persists |
| Short retention | Lower privacy risk | Less history for support/fraud |
| Long retention | Audit and analysis value | Greater exposure and cost |
| Immutable backups | Strong recovery | Delayed physical deletion |
| Crypto-shredding | Fast broad erasure | Key granularity and proof complexity |
| External processors | Capability and scale | Propagation and contract dependency |
| Rich AI traces | Debuggability | Sensitive-copy proliferation |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| No inventory or owner | Copies are missed | Catalog and lineage gates |
| Delete only primary row | Indexes and analytics retain data | Manifest-driven fan-out |
| Old event replays | Deleted record resurrects | Versioned tombstone |
| Cascade is too broad | Required records destroyed | Record-class ownership rules |
| Hold checked too late | Evidence deleted | Hold gate before execution |
| Cache invalidation fails | Residual exposure | ID-based purge plus bounded TTL |
| Object versions ignored | File remains recoverable | Enumerate and delete versions |
| Sensitive logs persist | Privacy incident | Allowlist logging and short retention |
| Backup restore skips tombstones | Data returns to production | Mandatory post-restore replay |
| Processor never acknowledges | Incomplete request | Tracked SLA, retry, escalation |
| Embedding deleted but chunk remains | Source still exposed | Lineage-based artifact purge |
| Source deleted but vector remains | Retrieval leakage | Index deletion verification |
| Training promise is vague | Unmet erasure expectation | Explicit pre-training policy |
| Identity verification is weak | Wrongful access or deletion | Risk-based verification |
| Tenant admin scope is assumed | Cross-tenant disclosure | Server-enforced authorization |
| Metrics contain deleted values | Shadow copy persists | Aggregate operation telemetry |

## What to measure

Track classified datasets with owners, lineage coverage, records past retention, hold propagation latency, deletion requests by state, oldest incomplete request, per-system completion and retry rate, processor acknowledgement latency, cache/index purge failures, resurrection detections, backup expiry horizon, restore-tombstone test results, sensitive-data logging violations, access-policy denials, AI artifacts missing lineage, and deletion verification sampling.

Measure data avoided as well as data deleted: unnecessary fields blocked at collection, prompt payload reduction, logs prevented, and expired datasets retired.

## Interview prompt

> Design privacy, retention, access, correction, and deletion controls for a multi-tenant AI operations platform. It stores construction documents and invoices, supports healthcare workflows, uses object storage, databases, caches, search/vector indexes, analytics, logs, backups, external processors, and model evaluation data.

Cover inventory and lineage, minimization, purpose-bound access, policy clocks, legal holds, deletion orchestration, tombstones, replicas and backups, AI derivatives and training, processor propagation, verification, audit, failure recovery, and tenant isolation.

## Worked answer

**One-line conclusion:** Make privacy an enforceable data lifecycle: minimize by purpose, attach lineage and retention to every copy, and execute deletion as an idempotent, verifiable distributed workflow.

“I would begin with a field-level inventory linking each source to every online, analytical, external, and AI derivative. Collection and access are purpose-scoped; tenant and subject context comes from trusted server state. Each record class has an owner, policy version, start trigger, due time, disposal action, and hold behavior.

A verified request creates a versioned deletion manifest. Before execution, the workflow checks authority, retention duties, and holds, then freezes new derivatives. It deletes authoritative data, publishes a payload-free tombstone with a deletion epoch, and fans out idempotent tasks to replicas, object versions, caches, search/vector indexes, analytics, logs where applicable, evaluation stores, and processors. Each target reports evidence or a bounded exception. Old events cannot outrank the tombstone.

Immutable backups remain isolated until expiry; if restored, current tombstones run before reopening service. AI artifacts carry source lineage, and training-data policy states honestly whether future-dataset exclusion, retraining, or another validated remediation is available. Audit stores minimal action facts rather than copied content. Construction records separate personal contact data from invoices and safety evidence; healthcare workflows preserve authorized clinical amendments and minimum-necessary access. Dashboards expose overdue retention, incomplete deletions, processor delays, resurrection, and restore-test failures.”

## Practice lab

A closed construction project contains a former subcontractor’s phone number in PostgreSQL, CRM, object metadata, OCR text, chunks, embeddings, logs, analytics, evaluation examples, and a notification provider. Invoice and safety records must remain under policy, an active claim places some correspondence on hold, immutable backups expire in 45 days, and an offline CRM connector may replay old data.

Design:

- the inventory and lineage identifiers;
- identity and authorization checks;
- record-class retention decisions;
- hold scope and precedence;
- deletion manifest and state machine;
- tombstone/version rules;
- live-store, vector, cache, log, processor, and backup treatment;
- AI training/evaluation policy;
- restore behavior;
- verification evidence and user-facing completion status.

Then adapt the design for a patient correction request where clinical history must be amended rather than overwritten.

## References

- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [NIST Privacy Framework 1.0 Core](https://www.nist.gov/system/files/documents/2020/01/16/NIST%20Privacy%20Framework_V1.0.pdf)
- [HHS — HIPAA Privacy Rule](https://www.hhs.gov/hipaa/for-professionals/privacy/index.html)
- [HHS — Guidance Regarding Methods for De-identification of PHI](https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html)
- [OWASP — Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [CISA — Data Security guidance](https://www.cisa.gov/topics/cyber-threats-and-advisories/data-security)
