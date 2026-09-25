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
- Next: caching and invalidation; queues and idempotency.

Each lesson includes a mental model, choices, failure modes, a worked example, interview practice, and references where useful. Daily updates should add a substantial lesson or refine an earlier one, update this index, and explain changes. Avoid invented performance claims.
