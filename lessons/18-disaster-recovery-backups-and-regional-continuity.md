# 18 — Disaster recovery, backups, and regional continuity

Previous: [Capacity planning, cost engineering, and multi-tenant fairness](17-capacity-planning-cost-engineering-and-multi-tenant-fairness.md). Study time: 50–60 minutes including the exercise.

## Mental model

High availability handles expected component failures while the system remains mostly intact. Disaster recovery (DR) restores an acceptable service after a site, region, account, control plane, dataset, credential, or operator boundary is lost or corrupted.

Start from business consequences:

- **RPO (recovery point objective):** maximum acceptable data loss measured in time;
- **RTO (recovery time objective):** maximum acceptable time to restore the capability;
- **WRT (work recovery time):** time after infrastructure returns to reconcile data, validate integrity, and resume business operations;
- **maximum tolerable downtime:** RTO plus business recovery must remain below it.

> Replication preserves continuity; independent recoverable history protects against corruption, deletion, ransomware, and operator error.

A replica can copy a bad delete immediately. A backup that has never been restored is only a hope. A region that can serve traffic but cannot authorize users, decrypt data, resolve DNS, access model artifacts, or reconcile external actions is not recovered.

## When to use each protection

| Mechanism | Protects against | Does not protect against alone |
| --- | --- | --- |
| Multi-zone deployment | Instance/zone failure | Regional or account failure |
| Cross-region replica | Regional loss; low RPO | Logical corruption copied to replica |
| Point-in-time recovery | Accidental writes/deletes | Missing external/object/event dependencies |
| Immutable backup | Ransomware/operator deletion | Fast application continuity |
| Object versioning | Overwrite/delete of objects | Database/workflow consistency |
| Event log/archive | Rebuild projections/indexes | Side effects already executed |
| Infrastructure as code | Environment reconstruction | Data, secrets, quota, DNS readiness |
| Warm standby | Faster regional recovery | Higher cost and drift |
| Multi-region active-active | Low RTO | Conflict, consistency, and operational complexity |
| Reconciliation ledger | External system divergence | Restoring internal data itself |

Layer protections according to failure modes and business objectives.

## Define recovery tiers

Not every capability needs the same objective.

| Tier | Example | RPO/RTO direction | Recovery posture |
| --- | --- | --- | --- |
| 0 | Identity, tenant policy, workflow truth, encryption keys | Lowest loss/time | Cross-region/warm controls, strict tests |
| 1 | Approval, operation ledger, reconciliation | Very low RPO | Durable replication plus immutable history |
| 2 | Documents and authoritative business data | Low RPO; moderate RTO | Versioned objects and database restore |
| 3 | Search indexes, embeddings, caches | Rebuildable | Restore source and regenerate |
| 4 | Analytics, experiments, temporary artifacts | Longer objectives | Recompute or accept loss |

Classify by user harm and reconstructability, not infrastructure preference. Record dependencies: a Tier 1 workflow is unusable if its Tier 0 authorization or keys are unavailable.

## Threat model

Plan for more than a cloud-region outage:

- database corruption or accidental delete;
- application bug writing bad data for hours;
- object overwrite or lifecycle misconfiguration;
- compromised administrator or ransomware;
- cloud account/control-plane lockout;
- key-management or identity-provider failure;
- DNS/certificate failure;
- event-stream loss or duplicate replay;
- external ERP/EHR state diverging from restored local state;
- model, prompt, tool schema, or embedding artifact unavailable;
- backup credential or encryption-key loss;
- correlated dependency/provider outage.

Each scenario has a different safe recovery point and containment step.

## Backup design

A backup policy defines:

```text
dataset and owner
business tier
RPO / RTO / retention
full, incremental, log, or snapshot method
frequency and geographic/account placement
encryption and key dependency
immutability/deletion controls
catalog and integrity verification
restore order and runbook
test frequency and evidence
legal hold/deletion obligations
```

Use the 3-2-1 idea as a prompt, not a magic rule: multiple copies, more than one failure boundary, and at least one isolated or immutable copy. Separate backup administration and credentials from production. Protect deletion with retention locks where appropriate, multi-party approval, alerts, and audited break-glass access.

Encrypt backups, but ensure keys and restoration identities survive the same disaster. Escrow or replicate key material according to policy; never put plaintext keys beside the backup.

## Application-consistent recovery

Crash-consistent storage is not necessarily business-consistent. A workflow may span:

- relational state;
- object documents;
- queue/event positions;
- search/vector indexes;
- approval and operation ledgers;
- external ERP/EHR actions.

Define authoritative sources and a recovery watermark. Preserve transaction IDs, object versions, event offsets, workflow versions, approval hashes, and external operation IDs.

Use transaction logs and point-in-time recovery for databases. Treat object versions as immutable inputs. Rebuild derived caches and indexes from authoritative data plus events. Do not restore a search index snapshot that refers to object versions newer than the recovered database without validation.

## Restore testing

Test the complete recovery path in an isolated environment:

1. select a point in time;
2. obtain backup catalog and keys through recovery identities;
3. restore authoritative stores;
4. verify checksums, row counts, constraints, tenant isolation, and time boundaries;
5. restore/replay events without executing external side effects;
6. rebuild derived indexes and caches;
7. deploy compatible application and AI bundle versions;
8. run business invariants and representative workflows;
9. measure actual RPO, infrastructure RTO, and work-recovery time;
10. preserve evidence and remediate gaps.

Restore tests must not send email, payments, ERP submissions, patient messages, or clinical orders. Use fenced networks and simulation adapters.

Monitor backup freshness and integrity, but do not confuse successful backup jobs with recovery evidence.

## Regional strategies

| Strategy | Cost | Typical recovery | Main complexity |
| --- | --- | --- | --- |
| Backup and restore | Lowest | Longest | Rebuild and data restore |
| Pilot light | Low–medium | Long | Scale minimal core |
| Warm standby | Medium–high | Shorter | Drift, capacity, data sync |
| Active-active | Highest | Potentially shortest | Writes, conflicts, fencing |

Choose per tier. A warm database with a cold authorization dependency still yields a cold service.

Pre-provision DNS, certificates, network routes, service identities, KMS access, quotas, images, model artifacts, observability, and support access. Infrastructure as code cannot create unavailable quota during the event.

## Failover and fencing

Failover is a state transition, not merely changing DNS:

1. declare and establish the last trustworthy state;
2. stop or fence the old writer;
3. measure replica lag and potential loss;
4. decide whether to promote, restore, or remain read-only;
5. promote one authoritative writer;
6. update routing with controlled TTL/health;
7. start consumers and schedulers without duplicates;
8. validate identity, policy, data, and external integration;
9. ramp traffic by capability;
10. reconcile missing and uncertain operations.

Use leases, epochs/fencing tokens, or storage-level writer controls to prevent split brain. Network isolation alone may fail when the old region reconnects.

If the recovery replica is behind, acknowledge the RPO loss explicitly. Do not accept new conflicting writes until missing operations are understood.

## Restore versus failover

Choose:

- **failover** when the standby state is trustworthy and sufficiently current;
- **point-in-time restore** when current replicas contain logical corruption;
- **read-only recovery** when the data-loss boundary is uncertain;
- **selective repair** when corruption scope is narrow and provable;
- **rebuild derived state** for indexes, caches, and projections;
- **manual reconciliation** for external side effects and ambiguous records.

Promoting a corrupted replica restores availability but preserves corruption. Restoring too far back may lose approvals and replay business decisions.

## Event replay and side effects

Event replay must distinguish state reconstruction from external actions.

Safe replay:

- rebuild search/vector indexes;
- recompute projections and analytics;
- regenerate derived metadata from immutable inputs.

Unsafe blind replay:

- submit invoice/payment;
- send email/SMS;
- place clinical order;
- notify external webhook;
- repeat a human approval transition.

Consumers use event IDs, idempotency records, operation ledgers, and replay mode. External action adapters must be disabled or reconcile by stable operation ID before any action.

## AI artifacts and recovery

Production AI depends on artifacts beyond model weights:

- model/provider and routing configuration;
- prompts and system instructions;
- tool schemas/adapters;
- workflow/state-machine definitions;
- policies and guardrails;
- evaluation suites and release evidence;
- embedding model and vector-index version;
- source-document lineage;
- tokenizer, runtime, container, and dependencies.

Store immutable versioned bundles in recoverable registries. A vector index is usually derived: recover documents, tenant/authorization metadata, chunking version, embedding version, and change log, then rebuild and atomically publish.

If the original model is unavailable, only use a pre-evaluated compatible fallback. Recovery urgency does not waive safety or tenant controls.

## Construction example

A regional database corruption is discovered two hours after a faulty migration. Both live replicas contain it, so ordinary failover is unsafe.

1. stop invoice approval and ERP writes; keep unaffected document reads clearly labeled;
2. identify the last clean database point and affected object/event ranges;
3. restore the database into the recovery region using isolated credentials;
4. restore immutable object versions and replay only state-building events;
5. deploy the matching application, prompt, workflow, policy, and tool bundle;
6. rebuild search/vector indexes from recovered authoritative sources;
7. compare approval hashes, operation ledgers, tenant counts, document versions, and business totals;
8. query ERP by stable operation IDs for the lost window;
9. reconstruct confirmed external outcomes without resubmitting them;
10. route read-only traffic first, then approvals, then ERP writes after fencing and validation.

The UI lists approvals in the uncertain window for controller review. Recovery is complete only when data and external ERP state agree or each exception has an owner.

## Healthcare variation

A corrupted clinical feed updates records in both regions. The team restores the last clean patient-data point, identifies encounters in the corruption window, and prevents synthesized discharge guidance until current source data is validated.

Patient identity, consent, break-glass rules, encryption keys, audit history, and source provenance are Tier 0 dependencies. Clinicians review affected encounters and determine whether drafts influenced care. Recovery artifacts containing PHI receive normal access, retention, and audit controls.

An available but incomplete patient record is not successful recovery.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Frequent backup/log shipping | Lower RPO | Cost and operational load |
| Long retention | More recovery points | Cost, privacy, deletion burden |
| Immutable isolated backup | Corruption/ransomware defense | Slower access and admin complexity |
| Cross-region replica | Low RPO/RTO | Copies logical corruption |
| Warm standby | Faster recovery | Cost and configuration drift |
| Active-active | Fast continuity | Conflict and split-brain complexity |
| Restore whole system | Simple concept | Slow, large blast radius |
| Selective repair | Less disruption | Hard proof and reconciliation |
| Rebuild index | Consistent lineage | Time and inference cost |
| Restore index snapshot | Fast | Version mismatch/staleness |
| Low DNS TTL | Faster routing | Resolver load and incomplete control |
| Read-only recovery | Protects integrity | Business delay |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Replica called a backup | Delete/corruption replicated | Immutable point-in-time history |
| Backup job green, restore untested | False confidence | Scheduled full restore drills |
| Backup shares production credentials | Attacker deletes both | Separate account/identity |
| Keys unavailable in DR | Encrypted data unusable | Tested key recovery |
| IaC without quotas/artifacts | Region cannot start | Pre-provision critical dependencies |
| Promote without fencing | Split brain | Epoch/lease writer fencing |
| DNS switched before validation | Users reach broken region | Dependency and business checks |
| Replay sends side effects | Duplicate payment/message | Replay mode and operation ledger |
| Index newer than source DB | Wrong/stale retrieval | Lineage watermark and rebuild |
| Old workflow bundle missing | State cannot resume | Retained compatible artifacts |
| RTO excludes reconciliation | Business still unusable | Measure WRT too |
| Restore violates deletion policy | Resurrected prohibited data | Deletion ledger and re-application |
| DR test uses real integrations | Production side effects | Fenced simulation environment |

## What to measure

Track backup success, age, completeness, immutability, and integrity; restore-test pass rate; measured RPO/RTO/WRT by tier; replication lag; recovery dependency readiness; key and identity recovery; data invariant failures; index rebuild progress; unresolved and duplicate external operations; failover/failback duration; capacity in recovery region; runbook drift; and recovery action ownership.

## Interview prompt

> Design disaster recovery for a multi-tenant AI assistant that stores construction documents, answers cited questions, reviews invoices, obtains approval, and submits to an ERP. A faulty migration corrupts both live database replicas, the primary region is unavailable, some ERP operations may have completed, and search indexes reference newer data. Adapt the design for a healthcare discharge assistant.

Cover recovery tiers, RPO/RTO/WRT, backup isolation, application consistency, restore testing, regional strategy, fencing, event replay, AI artifacts, external reconciliation, security/privacy, validation, and failback.

## Worked answer

**One-line conclusion:** Recover authoritative versioned truth from an independently tested point, fence all writers, rebuild derived AI state, and reconcile external effects before restoring consequential writes.

“I would tier data by business impact and define RPO, infrastructure RTO, and work-recovery time. Identity, tenant policy, keys, workflow truth, approvals, and operation ledgers receive the strongest cross-region and immutable-backup controls; search indexes and caches are rebuildable. Because corruption reached both replicas, I would stop writes and choose point-in-time restore rather than promote. Backups live in a separate protected boundary with tested recovery identities and keys. In the recovery region I restore authoritative database and object versions to a common watermark, deploy the matching application and AI bundle, replay only state-building events, and rebuild the vector index from document lineage. The old writer is fenced with an epoch before any promotion. ERP/email adapters remain disabled during replay. I reconcile the lost window by stable operation ID and record confirmed external outcomes without re-executing them. I validate tenant isolation, approvals, totals, object versions, citations, and SLOs; then ramp read-only, approvals, and writes separately. Actual RPO, RTO, reconciliation time, exceptions, and restore evidence feed the next DR test.”

## Practice lab

Respond to this scenario:

- database corruption began at 1:10 PM and was detected at 3:00 PM;
- cross-region replicas contain the corruption;
- transaction logs permit point-in-time restore every minute;
- object storage has immutable versions;
- 42 approvals and 18 ERP attempts occurred in the uncertain window;
- six ERP attempts timed out after send;
- the vector index includes document versions newer than the clean restore point;
- the primary region and its KMS control plane are unavailable.

Choose the recovery point, calculate the maximum potential loss, define key recovery and writer fencing, establish restore order, separate safe and unsafe event replay, rebuild the index, reconcile approvals/ERP actions, specify user-visible modes, and list the evidence required before writes resume.

## References

- [NIST SP 800-34 Rev. 1 — Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)
- [AWS Well-Architected — Disaster recovery options in the cloud](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/disaster-recovery-dr-objectives.html)
- [Google Cloud — Disaster recovery planning guide](https://cloud.google.com/architecture/dr-scenarios-planning-guide)
- [Kubernetes — Disaster recovery](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#disaster-recovery)
- [CISA — Ransomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide)
