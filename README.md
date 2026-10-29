# System Design

A daily learning path from core distributed systems to production AI, tailored to Forward Deployed Engineer and AI Engineer interviews.

## How to learn

1. Read a lesson's mental model and sketch the design before its worked answer.
2. Answer the interview prompt aloud in 3–5 minutes. Lead with the requirements and a clear decision.
3. Record your answer and questions in [the learning log](learning-log.md).
4. Revisit earlier components when a new lesson exposes a missing assumption.

The running example is a multi-tenant operations assistant: it reads authorized construction or healthcare records, returns cited answers, and proposes actions that require approval. Examples are exercises, not claims about a deployed product.

## Roadmap

| Stage | Components | Goal |
| --- | --- | --- |
| Foundations | requirements, estimates, latency budgets, APIs, data models | Bound the problem |
| Data path | load balancing, caching, queues, databases, indexes, object storage, search | Trace every request |
| Reliability | timeouts, retries, idempotency, backpressure, replication, consistency | Handle partial failure |
| Security | identity, authorization, tenant isolation, encryption, audit | Protect data and actions |
| AI systems | ingestion, retrieval, context, agent state, tools, evaluations, human approval | Build a grounded workflow |
| Operations | deployment, observability, rollouts, cost, incidents | Run and improve it |

### Lessons

- [01 — Requirements and capacity](lessons/01-requirements-and-capacity.md)
- [02 — API boundaries and request lifecycle](lessons/02-api-boundaries-and-request-lifecycle.md)
- [03 — Relational data modeling and indexes](lessons/03-relational-data-modeling-and-indexes.md)
- [04 — Caching and invalidation](lessons/04-caching-and-invalidation.md)
- [05 — Queues, delivery semantics, and idempotent consumers](lessons/05-queues-delivery-semantics-and-idempotent-consumers.md)
- [06 — Load balancing and stateless services](lessons/06-load-balancing-and-stateless-services.md)
- [07 — Object storage and safe document ingestion](lessons/07-object-storage-and-safe-document-ingestion.md)
- [08 — Search indexes and hybrid retrieval](lessons/08-search-indexes-and-hybrid-retrieval.md)
- [09 — Replication, consistency, and failover](lessons/09-replication-consistency-and-failover.md)
- [10 — Identity, authorization, and tenant isolation](lessons/10-identity-authorization-and-tenant-isolation.md)
- [11 — Encryption, secrets, and key management](lessons/11-encryption-secrets-and-key-management.md)
- [12 — Agent architecture, workflow state, and tool boundaries](lessons/12-agent-architecture-workflow-state-and-tool-boundaries.md)
- [13 — AI evaluations, guardrails, and release gates](lessons/13-ai-evaluations-guardrails-and-release-gates.md)
- [14 — Observability for distributed and AI workflows](lessons/14-observability-for-distributed-and-ai-workflows.md)
- [15 — Deployment strategies, feature flags, and safe rollouts](lessons/15-deployment-strategies-feature-flags-and-safe-rollouts.md)
- [16 — Incident response, graceful degradation, and recovery](lessons/16-incident-response-graceful-degradation-and-recovery.md)
- [17 — Capacity planning, cost engineering, and multi-tenant fairness](lessons/17-capacity-planning-cost-engineering-and-multi-tenant-fairness.md)
- [18 — Disaster recovery, backups, and regional continuity](lessons/18-disaster-recovery-backups-and-regional-continuity.md)
- [19 — Data contracts, schema evolution, and integration resilience](lessons/19-data-contracts-schema-evolution-and-integration-resilience.md)
- [20 — Human review systems, approval queues, and operator experience](lessons/20-human-review-systems-approval-queues-and-operator-experience.md)
- [21 — Privacy engineering, retention, and deletion workflows](lessons/21-privacy-engineering-retention-and-deletion-workflows.md)
- [22 — Configuration management, policy engines, and tenant customization](lessons/22-configuration-management-policy-engines-and-tenant-customization.md)
- [23 — Batch processing, stream processing, and change data capture](lessons/23-batch-stream-processing-and-change-data-capture.md)
- [24 — FDE discovery, integration architecture, and phased delivery](lessons/24-fde-discovery-integration-architecture-and-phased-delivery.md)
- [25 — End-to-end capstone: multi-tenant AI operations assistant](lessons/25-end-to-end-capstone-multi-tenant-ai-operations-assistant.md)
- [26 — System-design interview drills, scoring rubric, and answer refinement](lessons/26-system-design-interview-drills-scoring-and-refinement.md)
- [27 — Distributed rate limiting, quotas, and admission control](lessons/27-distributed-rate-limiting-quotas-and-admission-control.md)
- [28 — Distributed locks, leases, fencing tokens, and leader election](lessons/28-distributed-locks-leases-fencing-tokens-and-leader-election.md)
- [29 — Sharding, partitioning, rebalancing, and hot-key mitigation](lessons/29-sharding-partitioning-rebalancing-and-hot-key-mitigation.md)
- [30 — Service discovery, health checking, and resilient routing](lessons/30-service-discovery-health-checking-and-resilient-routing.md)
- [31 — Time, ordering, logical clocks, and causal consistency](lessons/31-time-ordering-logical-clocks-and-causal-consistency.md)
- [32 — Distributed transactions, sagas, compensation, and reconciliation](lessons/32-distributed-transactions-sagas-compensation-and-reconciliation.md)
- [33 — Load testing, chaos experiments, and performance validation](lessons/33-load-testing-chaos-experiments-and-performance-validation.md)
- [34 — AI inference serving, batching, caching, and GPU capacity](lessons/34-ai-inference-serving-batching-caching-and-gpu-capacity.md)
- [35 — Context engineering, prompt assembly, and token budgets](lessons/35-context-engineering-prompt-assembly-and-token-budgets.md)
- [36 — Model registries, artifact lineage, and reproducible AI releases](lessons/36-model-registries-artifact-lineage-and-reproducible-ai-releases.md)
- Next: multimodal document intelligence, OCR, layout, and evidence extraction.

Each lesson includes a mental model, choices, failure modes, a worked example, interview practice, and references where useful. Daily updates should add a substantial lesson or refine an earlier one, update this index, and explain changes. Avoid invented performance claims.
