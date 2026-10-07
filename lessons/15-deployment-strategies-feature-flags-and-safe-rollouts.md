# 15 — Deployment strategies, feature flags, and safe rollouts

Previous: [Observability for distributed and AI workflows](14-observability-for-distributed-and-ai-workflows.md). Study time: 50–60 minutes including the exercise.

## Mental model

A deployment moves artifacts into an environment. A **release** changes what users or workflows experience. Separating the two lets you install code safely, validate it, and expose behavior gradually.

For production AI, the release unit is not only a container:

```text
release bundle =
  application + model route + prompt + tool schemas + workflow definition
  + retrieval index/embedding space + policy + feature configuration
  + database/event compatibility
```

Every bundle needs an immutable version, compatibility contract, evaluation result, rollout policy, owner, and rollback path.

The governing rule is:

> Make changes backward-compatible first, expose them to a small stable cohort, measure user and safety outcomes, and preserve a tested path to stop or reverse exposure.

Rollback is not always “deploy the old binary.” Database writes, queue messages, external actions, re-embedded documents, and long-running workflows may outlive the code that created them.

## Deployment versus release controls

| Mechanism | Controls | Best use | Limitation |
| --- | --- | --- | --- |
| Rolling deployment | Which instances run a binary | Routine stateless service updates | Old and new coexist during rollout |
| Blue-green | Which complete environment receives traffic | Fast switch and infrastructure rollback | Expensive; state is still shared |
| Canary | Which cohort receives a version | Measure risk on real traffic | Needs stable routing and automated gates |
| Feature flag | Which behavior executes | Decouple code deploy from exposure | Flag debt and inconsistent combinations |
| Shadow mode | Candidate processes copied inputs | Validate performance and outputs without exposure | Must suppress side effects |
| A/B experiment | Which experience users receive | Compare product outcomes | Not appropriate for unbounded safety risk |
| Ring deployment | Which risk tier receives change | Internal → pilot → low-risk → broad rollout | Slower and needs representative rings |
| Kill switch | Disable a capability quickly | Stop tools, model route, ingestion, or writes | Must be tested and fail safely |

These mechanisms compose. A new build can be deployed rolling, exercised in shadow mode, released by a tenant-sticky flag to a canary ring, and protected by a write-tool kill switch.

## Classify the change first

The rollout should match the failure radius.

| Change | Main risks | Required compatibility |
| --- | --- | --- |
| Stateless API code | Crashes, latency, contract changes | Old/new clients and servers |
| Database schema | Locking, irreversible writes, mixed versions | Expand-contract and backfill |
| Event schema | Old consumers fail or misinterpret | Additive evolution and tolerant readers |
| Workflow definition | In-flight state cannot resume | Version pinning and migration |
| Prompt/model | Quality, cost, latency, safety regression | Output/tool schema and evaluation |
| Embedding/index | Mixed vector spaces, stale results | Dual write/reindex and atomic alias |
| Authorization policy | Overexposure or denial | Default-deny and policy-version behavior |
| External connector | Duplicate/unknown side effects | Idempotency and reconciliation |

Do not choose “10% canary” mechanically. One percent of requests can still include the highest-value tenant or a harmful write path. Define exposure in units of tenants, workflows, risk classes, and consequential actions.

## Build immutable, promotable artifacts

Build once and promote the same signed artifact across environments. Record:

- source commit and reproducible build;
- dependency lockfile and software bill of materials;
- container or package digest;
- migration and event-schema versions;
- model/provider identifier and routing configuration;
- prompt, tool, policy, workflow, and evaluation-suite versions;
- retrieval index and embedding versions;
- approvals and release notes.

Do not rebuild “the same” release separately for production. Mutable tags such as `latest` make rollback and incident reconstruction ambiguous.

Environment-specific configuration should be validated, secret references should be resolved at runtime, and production credentials must never be copied into test.

## Pre-production gates

Before exposure:

1. run unit, contract, integration, migration, security, and evaluation suites;
2. test mixed-version compatibility;
3. validate observability, dashboards, and deployment markers;
4. exercise rollback, kill switches, and data recovery;
5. load-test realistic request classes and dependency limits;
6. confirm capacity headroom for duplicate blue-green or shadow traffic;
7. record owners, stop conditions, and the rollout plan.

For AI changes, compare the whole candidate bundle with the baseline on identical cases. Hard gates cover tenant leakage, unauthorized tools, unsupported consequential claims, and duplicate actions. Quality, latency, and cost use slice-aware thresholds.

Passing offline tests makes a version eligible for controlled exposure; it does not prove production safety.

## Rolling deployments

A rolling deployment gradually replaces instances. It is efficient, but old and new versions coexist.

Requirements:

- readiness gates traffic only after dependencies and caches are ready;
- liveness detects deadlock without restarting merely slow healthy work;
- startup probes protect long initialization;
- connection draining stops new requests and allows bounded completion;
- termination handles streams, leases, and queue messages safely;
- max unavailable/max surge preserve capacity;
- APIs, database schema, caches, and messages work across adjacent versions.

Long LLM streams and model loading complicate draining. Stop admission, let short streams finish, checkpoint durable work, and enforce a deadline. Never kill a worker after an external request without recording whether the outcome is known.

## Blue-green deployment

Blue-green keeps old and new environments available and switches traffic after validation. It supports fast traffic rollback, but both environments usually share databases, queues, object storage, and external systems.

Therefore:

- schema changes must remain backward-compatible;
- background consumers must not double-process the same queue;
- scheduled jobs need leader election or environment fencing;
- cache namespaces may need versioning;
- side effects must remain idempotent;
- capacity and cost roughly double during overlap.

Switching traffic back does not undo writes made by green. Treat traffic rollback and data recovery as separate operations.

## Canary and ring rollouts

A canary exposes a candidate to a small, controlled cohort. Route deterministically:

```text
cohort = hash(tenant_id + experiment_salt) mod 10_000
```

Tenant-sticky routing keeps users, cached state, and multi-step workflows consistent. For durable workflows, pin the bundle version when the workflow begins; do not switch model, prompt, tool schema, or policy midway unless an explicit compatible migration exists.

A practical ring sequence:

1. synthetic and internal traffic;
2. employees/test tenants;
3. named pilot tenants with support coverage;
4. low-risk read-only workflows;
5. broader reads;
6. human-approved writes;
7. general availability.

Each promotion requires a minimum sample or time window, healthy SLO burn, hard safety gates, quality checks, capacity checks, and an accountable decision. Increase exposure only when the current ring provides enough evidence.

## Shadow mode

Shadowing sends copied production inputs to the candidate while users remain on the baseline.

Safe shadow adapters must:

- replace writes with recorded simulations;
- block emails, payments, ERP mutations, and patient communication;
- use isolated caches and temporary state;
- respect production authorization, consent, and retention;
- avoid doubling load beyond dependency budgets;
- identify candidate output for offline comparison.

Do not rely on a prompt saying “do not execute.” Enforce side-effect suppression in code and credentials.

Shadowing reveals latency, compatibility, and output differences. It cannot measure user reaction or the real consequences of actions.

## Feature flags

Flags should be typed, owned, observable, and temporary.

A flag definition includes:

```text
flag_key
purpose and owner
allowed values
default and fail behavior
targeting dimensions
created/expires_at
prerequisites and conflicts
audit history
removal condition
```

Evaluate flags from trusted server-side attributes. Never let a client assert its tenant tier or authorization. Cache configuration with bounded staleness and define behavior when the flag service is unavailable.

Separate:

- **release flags** for temporary rollout;
- **experiment flags** for randomized comparison;
- **operational flags** or kill switches;
- **entitlement flags** for contracted capability.

Entitlements are authorization inputs, not a substitute for authorization. A flag enabling `submit_invoice` does not grant the user permission to submit one.

Every flag combination expands the test matrix. Limit dependencies, document precedence, emit flag decisions in traces, and remove completed release flags promptly.

## Database expand-contract

Old and new application versions must work during rollout:

1. **Expand:** add nullable columns, new tables, or additive indexes without breaking old code.
2. **Migrate:** deploy code that can read old/new forms; dual-write only when necessary and observable.
3. **Backfill:** process in bounded resumable batches with checkpoints and throttling.
4. **Verify:** compare counts, constraints, hashes, and business invariants.
5. **Switch:** change reads using a flag after verification.
6. **Contract:** remove old fields only after all readers, writers, jobs, and rollback windows have moved.

Avoid long blocking migrations in the release path. A successful schema rollback is often impossible once new-format data is written; forward-fix or compatibility mode may be safer.

Dual writes can diverge. Prefer one transactional source of truth plus an outbox when crossing systems. Measure mismatch and provide reconciliation.

## Events, APIs, and cache compatibility

Evolve events additively:

- consumers ignore unknown optional fields;
- producers preserve required semantics;
- new required fields receive defaults or a new version;
- schema compatibility is checked in CI;
- replay tests cover older retained messages;
- poison events go to a DLQ with safe repair.

During API evolution, add fields before removing them, version incompatible semantics, and deploy tolerant consumers before new producers.

Version cache keys when representation semantics change. Otherwise a new writer may store data an old reader cannot interpret. Plan for cache warming without stampeding dependencies.

## AI release bundles

Model, prompt, retrieval, and tools interact. Treat these combinations as tested bundles:

```text
ai_bundle_v18
  model_route: reasoning-v5
  prompt: invoice-review-23
  embedding/index: embed-v4 / invoice-index-v4
  reranker: rr-7
  tool_schema: invoice-tools-v6
  policy: finance-policy-12
  workflow: invoice-review-v9
```

Record the resolved bundle on every workflow. A dynamic model router must emit the actual model used, not only the intended route.

For an embedding migration, build a separate index, dual-publish document updates, backfill, validate retrieval and authorization, compare in shadow, and atomically switch an alias. Never query vectors from incompatible spaces together.

Prompt changes can alter tool arguments and approval previews. Re-run contract and safety evaluations even when application code does not change.

## Long-running workflows

A workflow may last longer than the deployment. Choose explicitly:

- **pin:** old workflows finish on their original definition;
- **compatible resume:** new workers understand old state/schema;
- **migrate:** a versioned, idempotent state transformation upgrades selected workflows;
- **drain:** stop starting old workflows and wait for completion;
- **cancel/restart:** only when business semantics and side effects make it safe.

Store workflow version, state version, completed steps, plan/approval hash, tool versions, and operation IDs. Never replay an external side effect merely because a new worker cannot read the old checkpoint.

Policy changes require special handling. A critical revocation may need execution-time enforcement across old workflows, while a presentation-only change can remain pinned.

## Automated analysis and rollback

Compare candidate with baseline by request class and risk slice:

- error-budget burn and terminal success;
- latency, queue age, saturation, and dependency errors;
- groundedness, corrections, abstention, and guardrail events;
- tool denials, duplicate suppression, and uncertain outcomes;
- cost per successful task;
- tenant, language, document, and workflow slices.

Predefine stop thresholds. On violation:

1. freeze promotion;
2. disable the risky flag or tool;
3. route new work to the baseline;
4. decide whether in-flight workflows remain pinned, pause, or migrate;
5. reconcile uncertain side effects;
6. preserve evidence and open the incident process.

Automatic rollback is appropriate for clear, fast, reversible technical regressions. Security events and ambiguous business-side effects often require automatic containment plus human-led reconciliation.

## Construction example

The team releases a new invoice-review bundle with a better model, prompt, embedding index, and ERP connector.

1. CI evaluates the immutable bundle and checks schema/tool compatibility.
2. The new vector index is backfilled while document updates publish to both indexes.
3. Application code and an additive database column are deployed with the feature off.
4. Shadow traffic compares retrieval and proposed actions; ERP writes use a simulation adapter.
5. Internal and pilot tenants receive tenant-sticky cited review, but existing workflows remain pinned.
6. The canary enables human-approved ERP submission for a small low-risk cohort.
7. Dashboards compare grounding, exception recall, correction, latency, cost, tool errors, and uncertain outcomes.
8. A connector timeout regression trips the stop threshold. The write flag is disabled immediately, new work returns to the prior connector, and already-sent operations reconcile by stable operation ID.
9. The database column and v4 index remain because they are backward-compatible; no destructive rollback is needed.

The UI shows affected operations as “outcome unknown—reconciling,” not failed. Release rollback does not erase business reality.

## Healthcare variation

A discharge-planning candidate begins in advisory shadow mode. It cannot place orders, message patients, or modify the record. Clinical experts review critical slices: medication conflicts, allergies, pediatrics, pregnancy, corrected results, language, and ambiguous identity.

The canary is sticky by authorized care context, starts with selected units, and pins in-progress encounters to a validated bundle. Hard gates stop cross-patient data, unsupported medication changes, or bypassed clinician approval. A clinical kill switch disables recommendations while preserving ordinary record access.

Database, model, and evaluation artifacts containing PHI retain healthcare access, audit, and retention controls. “Test traffic” is not exempt from privacy requirements.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Rolling | Resource-efficient | Mixed-version window |
| Blue-green | Fast traffic switch | Double capacity; shared-state risk |
| Canary | Limits blast radius | Operational and statistical complexity |
| Shadow | No user-visible output | Duplicate load; no real action outcome |
| Feature flag | Decoupled exposure | Flag debt and state explosion |
| Tenant-sticky routing | Workflow consistency | Uneven cohorts |
| Request-level randomization | Balanced experiment | Breaks sessions and durable workflows |
| Pin old workflows | Predictable resume | Runs old code longer |
| Migrate workflows | Faster convergence | Migration correctness risk |
| Expand-contract | Safe mixed versions | Slower cleanup |
| Destructive migration | Simpler final schema | Weak rollback and downtime risk |
| Automatic rollback | Fast containment | Can worsen stateful failures |
| Manual approval | Context-aware decision | Slower response |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Treat deploy as release | All users exposed immediately | Flags and staged cohorts |
| Canary sampled per request | Workflow changes version midway | Tenant/workflow stickiness |
| Roll back binary after new writes | Old code cannot read state | Expand-contract and compatibility |
| Blue and green both consume queue | Duplicate side effects | Consumer fencing/idempotency |
| Shadow has production credentials | Real side effects | Simulation adapters and denied writes |
| Flag used as authorization | Unauthorized capability | Server-side policy check |
| Flag service outage flips behavior | Surprise broad exposure | Safe defaults and cached config |
| Untested flag combinations | Emergent failures | Limit dependencies and test matrix |
| Embedding spaces mixed | Invalid similarity results | Separate versioned indexes |
| Prompt updated without tool tests | Invalid or unsafe calls | Bundle contract evaluation |
| Old checkpoint resumed by incompatible worker | Corrupt transition | Pin or versioned migration |
| Auto rollback repeats ERP request | Duplicate invoice/payment | Stable operation ID and reconciliation |
| No minimum canary window | Promotion on weak evidence | Sample and time gates |
| Release flag never removed | Permanent complexity | Owner, expiry, cleanup |

## What to measure

Track:

- artifact and resolved bundle version by request/workflow;
- deployment readiness, restarts, drain time, and capacity;
- cohort exposure and flag evaluation;
- SLO burn, errors, latency, saturation, and queue age;
- workflow completion, state age, retries, and reconciliation;
- AI groundedness, corrections, guardrails, abstention, cost, and latency by version and slice;
- tool denials, invalid arguments, duplicates, and outcome-unknown rate;
- schema backfill progress, mismatch, lag, and invariant violations;
- promotion, pause, rollback, kill-switch, and override events;
- time to detect, contain, reconcile, and recover.

## Interview prompt

> Design a safe deployment and release process for a multi-tenant AI assistant that ingests construction documents, answers cited questions, reviews invoices, obtains approval, and submits to an ERP. The application, schema, prompt, model, embedding index, tools, policy, and workflow definitions change independently, while workflows may last hours. Adapt the design for healthcare discharge planning.

Cover change classification, immutable artifacts, compatibility, rolling/blue-green/canary choices, shadowing, feature flags, databases/events, workflow pinning, AI bundles, gates, observability, rollback, and uncertain side effects.

## Worked answer

**One-line conclusion:** Deploy immutable backward-compatible bundles first, release them through sticky risk-based rings, and roll back exposure without replaying or denying the real state of in-flight side effects.

“I would version the complete application-model-prompt-index-tool-policy-workflow bundle and promote the same signed artifacts across environments. CI runs contract, migration, security, load, and slice-aware AI evaluations, including mixed-version tests. Database and event changes use expand-migrate-contract so old and new instances coexist. Code deploys with behavior disabled. The candidate then processes write-suppressed shadow traffic, followed by internal, pilot, read-only, and human-approved-write rings. Routing is tenant-sticky, and each durable workflow stores its resolved bundle version. Old workflows remain pinned unless a tested idempotent migration exists. Flags are typed, server-evaluated, observable, owned, and temporary; entitlements never replace authorization. Promotion requires minimum evidence, healthy SLO burn, hard safety gates, and quality, latency, and cost thresholds. A tool kill switch can stop writes independently. On regression, I route new work to the baseline, preserve compatible schema, pause or pin in-flight work, and reconcile possibly accepted ERP operations using stable operation IDs. Traffic rollback never pretends that external writes were undone.”

## Practice lab

Design the rollout for this release:

- a new prompt emits tool schema v6;
- embedding v4 requires a new index;
- a nullable database column supports a new approval reason;
- workflows can run for six hours;
- the ERP connector changes retry behavior;
- the first pilot shows better grounding but twice as many outcome-unknown writes.

Specify the immutable bundle, compatibility sequence, shadow protections, routing unit, rollout rings, flag types, workflow pin/migration rule, promotion gates, kill switch, rollback actions, and reconciliation plan. Explain which artifacts may remain after rollback and why.

## References

- [Google SRE Workbook — Canarying Releases](https://sre.google/workbook/canarying-releases/)
- [Kubernetes — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes — Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [OpenFeature specification](https://openfeature.dev/specification/)
- [Martin Fowler — Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [AWS Builders' Library — Ensuring rollback safety during deployments](https://aws.amazon.com/builders-library/ensuring-rollback-safety-during-deployments/)
