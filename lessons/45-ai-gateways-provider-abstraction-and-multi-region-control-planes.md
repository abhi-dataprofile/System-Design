# AI gateways, provider abstraction, and multi-region control planes

An AI gateway is the governed boundary between applications and model-serving backends. It authenticates callers, applies policy, selects an eligible route, normalizes supported semantics, protects credentials and budgets, and records what happened.

It is not merely a reverse proxy. Ordinary proxies understand hosts, paths, bytes, timeouts, and status codes. An AI gateway must also understand model capabilities, tokens, streaming, tools, safety classifications, data residency, prompt sensitivity, evaluation state, and the risk of silently changing model behavior.

## Mental model

Separate the system into three planes:

- **request data plane:** handles each inference request and stream;
- **control plane:** defines models, providers, policies, budgets, regions, credentials, releases, and routing rules;
- **evidence plane:** stores privacy-safe traces, usage, costs, evaluation results, incidents, and decision lineage.

The data plane should be thin, deterministic, bounded, and available. The control plane can be richer but must publish immutable, validated configuration snapshots. The evidence plane makes every route and fallback explainable.

A request becomes:

[
Route = f(identity, tenant, purpose, dataClass, capability, region, risk, budget, health, release)
]

Latency or price alone must not choose a route. A cheaper provider is ineligible if it lacks the required tool semantics, regional approval, context length, evaluation gate, or contractual data controls.

The gateway owns **selection and enforcement**, not business truth. It should not decide whether an invoice is valid or whether a patient should receive care.

## When to use it

Use an AI gateway when several applications or tenants need one or more of:

- multiple model providers, deployments, or regions;
- centralized credentials and provider-account isolation;
- tenant quotas, token budgets, and cost attribution;
- data-residency or regulated-data routing;
- consistent observability and audit;
- provider failover and circuit breaking;
- controlled model aliases and release policies;
- request classification, redaction, or content safeguards;
- approved tool-calling schemas;
- gradual migration without changing every application.

A direct provider SDK is often better for a small experiment with one model and one team. Introduce a gateway only when the governance and operational value exceeds the additional latency, complexity, and blast radius.

## Start with a semantic contract

Do not promise that all providers are interchangeable. Define a canonical request envelope with explicit, versioned semantics:

- tenant, user/service identity, and purpose;
- workload class: interactive, batch, embedding, extraction, or agent step;
- requested capability, not only a model name;
- input messages or multimodal parts;
- tool definitions and allowed tool choice;
- structured-output schema;
- latency deadline and cancellation token;
- maximum input/output tokens;
- data classification and residency;
- risk tier and automation boundary;
- idempotency/request ID;
- trace context;
- application and release version;
- optional provider-specific extension under policy.

The response should include:

- generated content or structured result;
- normalized finish reason;
- selected provider, deployment, model revision or alias observation;
- route-policy and configuration revision;
- input/output/cached token accounting when available;
- latency phases;
- safety or moderation signals with provider-specific provenance;
- tool-call identifiers and arguments;
- truncation, fallback, and degradation indicators;
- error classification;
- evidence/tracing identifiers.

Normalization must not fabricate equivalence. If a provider does not expose cached tokens or safety categories, record “not reported,” not zero.

## Capability registry

Maintain a versioned registry for every deployable route:

- modality support;
- context and output limits;
- streaming behavior;
- tool-calling and parallel-tool semantics;
- structured-output guarantees;
- determinism controls;
- supported regions and accounts;
- data-use and retention contract;
- approved data classifications;
- model/release identity;
- evaluation status by workload and slice;
- latency and capacity envelope;
- pricing version and token accounting;
- known limitations;
- deprecation date.

Applications ask for a capability profile such as “authorized document extraction with JSON schema in US East,” not “whatever model is cheapest.”

Provider documentation is an input, not proof. Verify capabilities with contract tests and task evaluations.

## Data plane request lifecycle

A safe path is:

1. authenticate workload identity;
2. derive tenant and purpose from trusted credentials;
3. validate schema, size, modality, and deadline;
4. classify the request using trusted metadata and deterministic rules;
5. authorize data class, model capability, tools, and region;
6. reserve tenant/provider budget and admission capacity;
7. load one immutable routing-policy revision;
8. filter routes by hard eligibility constraints;
9. score eligible routes by release policy, health, quality, latency, and cost;
10. acquire scoped provider credentials;
11. transform the request with a versioned adapter;
12. send it with bounded timeout and cancellation;
13. stream through bounded buffers if requested;
14. normalize the response without hiding lost semantics;
15. settle token/cost usage and release reservations;
16. emit privacy-safe evidence and route outcome;
17. reconcile unknown provider outcomes and billing discrepancies.

Record the chosen configuration revision before dispatch. A mid-request control-plane change must not partially alter the route.

## Hard constraints before optimization

Separate **eligibility** from **preference**.

Hard constraints may include:

- tenant authorization;
- allowed provider account;
- region and residency;
- PHI/PII/confidential-data approval;
- model evaluation gate;
- required modality or tool feature;
- context/output limit;
- maximum risk tier;
- contractual retention mode;
- customer-managed-key requirement;
- available quota and deadline feasibility.

Only after filtering may the gateway optimize:

- expected task quality;
- latency;
- marginal cost;
- capacity;
- cache locality;
- provider diversification;
- energy or placement preferences.

This prevents a weighted score from trading away a non-negotiable policy.

## Provider adapters

Each adapter translates the canonical envelope to one provider API and back.

Test:

- role/message mapping;
- multimodal encoding;
- token-limit semantics;
- stop/finish reasons;
- tool-call names, IDs, arguments, and ordering;
- structured output and schema failures;
- streaming event order;
- cancellation behavior;
- timeouts and transport errors;
- usage accounting;
- safety refusal representation;
- context truncation;
- model alias/revision metadata.

A provider-specific escape hatch may be necessary. Namespace it, authorize it, log it, and accept that it reduces portability.

Avoid a lowest-common-denominator interface. It blocks useful capabilities while still failing to guarantee equivalence.

## Streaming and backpressure

Streaming creates a long-lived distributed request:

- client disconnect may not cancel provider generation;
- slow clients can fill gateway memory;
- first bytes may escape before a downstream safety check finishes;
- provider errors may occur after partial output;
- retries after emitted content can duplicate or contradict text;
- tool calls can arrive incrementally.

Use bounded buffers, idle and total deadlines, cancellation propagation, per-tenant concurrent-stream limits, and explicit partial-result status.

Once output is visible, do not transparently restart with another model. End the stream with a truthful error or let the application offer an explicit retry.

For buffered high-risk responses, validate the complete output before release, accepting higher time-to-first-token.

## Retries, hedging, and fallback

### Retry

Retry only errors likely to be transient and only when the request remains safe:

- use a total deadline and small attempt budget;
- honor provider retry guidance;
- apply jittered backoff;
- reserve quota for retry amplification;
- do not retry invalid input or policy rejection;
- do not blindly retry after partial streaming;
- track attempt lineage.

Text generation is usually read-like but not consequence-free: it consumes money, may create provider-side logs, and can return different content.

### Hedging

Sending the same request to two providers may reduce tail latency, but it doubles cost and data exposure and complicates cancellation. Permit hedging only for approved data classes and workloads, with delayed hedge start and loser cancellation. Record both dispatches.

### Fallback

Fallback is a product and safety decision, not merely availability logic.

A fallback model may change:

- answer quality and calibration;
- tool-call syntax;
- context capacity;
- refusal behavior;
- latency and cost;
- language or image support;
- contractual data handling.

Pre-evaluate each route chain. Mark fallback in the response. For high-risk extraction or action proposals, abstention or queueing may be safer than an unvalidated fallback.

## Tool calling and actions

The gateway can validate tool schemas and model eligibility, but the application’s durable workflow owns tools and side effects.

Enforce:

- tool allowlist by tenant, purpose, and risk;
- schema size and argument validation;
- maximum calls, depth, and parallelism;
- tool-call ID uniqueness;
- untrusted-output treatment;
- no provider-held business credentials;
- approval and revalidation outside the model;
- idempotency and outcome reconciliation at the tool boundary.

A gateway must never turn a model-generated function call directly into an ERP, building-control, or clinical operation.

## Credentials and provider accounts

Applications authenticate to the gateway with workload identity. They should not receive provider API keys.

Use:

- secret manager or KMS-backed storage;
- short-lived credentials where supported;
- separate provider accounts/projects by environment and risk domain;
- regional credentials;
- least privilege;
- automated rotation;
- egress controls;
- usage anomaly detection;
- emergency disable without redeploying clients.

A single shared key makes attribution, revocation, and blast-radius control difficult.

Do not log provider keys, authorization headers, raw signed URLs, or full confidential prompts.

## Budgets, quotas, and admission

Reserve expected work before dispatch using token estimates or workload weights.

Apply hierarchical limits:

- platform;
- provider/account;
- region;
- tenant;
- application;
- workload class;
- user or service.

Protect reconciliation, safety, and interactive workloads from bulk jobs. Separate request rate, concurrent streams, tokens, images/audio duration, and monetary budgets.

Estimate first, settle with reported usage, and reconcile later against provider billing. Provider tokenizers and accounting may differ, so normalized cost is an estimate until settlement.

A budget failure should be explicit. Do not silently send a request to a lower-quality route unless the application policy approves that tradeoff.

## Caching

Gateway caching is dangerous when identity, context, policy, or freshness changes the answer.

If used, include in the key:

- tenant and authorization scope;
- normalized input digest;
- model/provider release;
- system prompt and tool schema versions;
- policy/configuration revision;
- retrieval/source revision;
- safety and output parameters;
- data classification.

Do not cache personalized, time-sensitive, or action-bound output without a justified freshness and invalidation model. Encrypt and tenant-scope storage. A semantic cache has approximate matching risk and needs evaluation for false reuse.

Provider prompt caching is different: it can reduce processing for a shared prefix but still requires privacy, accounting, and version controls.

## Control plane

The control plane manages:

- provider and deployment registry;
- capability evidence;
- model aliases;
- adapter versions;
- tenant policy;
- regional eligibility;
- budget rules;
- routing and fallback chains;
- evaluation gates;
- credentials references;
- rollout state;
- emergency controls.

Publish signed or integrity-checked, immutable bundles with:

- monotonically increasing revision;
- schema version;
- creation and activation time;
- author/approver;
- source commit;
- dependency versions;
- compatibility constraints;
- rollback target.

Validate bundles before publication. Data planes should keep the last known good bundle and reject incompatible updates.

For a safety-critical deny or credential revocation, support a narrow emergency channel with stronger audit. Do not make every request synchronously depend on a remote control-plane database.

## Multi-region architecture

A practical design has regional data planes and a logically global control plane.

### Regional data plane

Each region contains:

- gateway instances across failure zones;
- regional policy/config cache;
- regional rate and concurrency enforcement;
- provider adapters and egress;
- regional secrets references;
- local telemetry buffer;
- optional tenant-safe cache.

Keep prompt and response data in the approved region. Metadata replication must also respect residency.

### Global control plane

The global service coordinates registry, releases, organization-wide policy, aggregate budgets, and provider health. It publishes region-specific bundles instead of serving every request.

Define a home region per tenant or data class. Route at the edge using trusted tenancy metadata, not prompt contents.

### Failover policy

Regional failover is eligible only if:

- destination is authorized for the tenant/data;
- credentials and provider contract are approved;
- required model/tool capabilities exist;
- configuration revision is compatible;
- data dependencies are accessible lawfully;
- latency/deadline remain feasible.

If not, fail closed, queue approved asynchronous work, or offer a reduced feature. Availability does not override residency.

### Global quotas

Strong global counters add latency and can fail during partitions. Options:

- allocate regional token/monetary leases from a global budget;
- use conservative regional allowances;
- reconcile asynchronously;
- reserve a protected emergency pool;
- stop allocating when leases expire.

This bounds overspend without a cross-region transaction on every request. During partition, each region may spend only its valid lease.

## Provider health

Measure health by route and workload, not provider name alone:

- DNS/TLS/connect latency;
- time to first token;
- inter-token delay;
- completion latency;
- transport and status errors;
- throttling;
- malformed streams;
- tool/schema validity;
- truncation and refusals;
- usage-report completeness;
- sampled quality and safety canaries.

A provider can be transport-healthy but semantically degraded.

Use circuit breakers with minimum sample sizes, half-open probes, and route-specific thresholds. Avoid synchronized flapping across regions. Manual override needs expiration and audit.

Synthetic prompts must contain no sensitive production data and should test capabilities without polluting downstream analytics.

## Observability and evidence

Create one trace across client, gateway, provider attempt, stream, and application workflow.

Record:

- trusted tenant/application identity;
- request and attempt IDs;
- route decision and rejected candidates with reason codes;
- configuration, adapter, prompt, tool-schema, and model versions;
- provider region/account/deployment;
- timing phases and retry/fallback;
- token/cost estimates and settlement;
- cache behavior;
- safety and schema outcomes;
- cancellation and partial-output state;
- prompt/response digest or approved redacted sample;
- evaluation and release cohort.

Do not make raw prompt logging the default. Use structured metadata, keyed digests, redaction, sampling, access controls, and short retention based on purpose.

Correlate gateway success with application outcomes. A 200 response containing an invalid tool call is not success.

## Release and evaluation

Treat every route as a release bundle:

- provider/deployment/model;
- adapter;
- canonical contract version;
- prompt/tool schema compatibility;
- data policy;
- fallback chain;
- evaluation evidence;
- rollout cohort.

Test provider adapters with recorded synthetic contracts and live canaries. Evaluate each workload and important slice: document type, language, handwriting, long context, tool use, structured output, and high-risk refusal.

Use shadowing only when duplicate data disclosure is approved. Shadow outputs must never trigger tools. Compare quality, schema validity, latency, cost, refusal, and safety before canary promotion.

A provider may change a floating alias. Detect observed model metadata and behavior drift; pin releases where possible and re-evaluate changes.

## Construction operations example

A multi-tenant construction assistant answers questions over drawings and contracts, extracts invoices, drafts RFIs, and proposes approved ERP actions. It uses a fast text model, a multimodal model, and a private regional deployment.

Requirements:

- project data must remain tenant-scoped;
- certain owners require US-only processing;
- drawing analysis requires vision;
- ERP proposals require a validated structured-output route;
- interactive questions have a short deadline;
- month-end extraction is batchable;
- one provider may throttle;
- applications must not store provider keys.

Design:

1. applications send a capability request with tenant, project, purpose, data class, deadline, and risk;
2. gateway derives identity and authorization from workload credentials;
3. policy filters providers by tenant contract, US region, modality, evaluation gate, context, and tool schema;
4. admission reserves tokens and provider quota by workload;
5. route selection chooses among eligible releases;
6. adapter translates the versioned request;
7. evidence records route, source/prompt versions, usage, and fallback;
8. interactive reads may use an evaluated fallback;
9. invoice extraction queues rather than using an unvalidated text-only fallback;
10. any ERP proposal returns to a durable approval workflow that revalidates source versions and executes idempotently;
11. provider throttling opens a route-specific circuit;
12. regional failure uses another US region only when the tenant policy, dependencies, and release permit it.

If a multimodal provider fails after streaming a partial drawing interpretation, the gateway stops with partial status. It does not append a second model’s answer invisibly. The application can preserve the partial evidence and offer an explicit retry.

## Healthcare operations variation

A healthcare operations assistant summarizes authorized operational records and extracts prior-authorization packet fields. Some requests contain PHI.

The gateway routes PHI only to explicitly approved accounts and regions with required contractual and retention settings. Workload identity, organization, purpose, and data classification are trusted metadata; the model cannot weaken them.

Raw prompts are not logged by default. Regional evidence contains protected metadata with short retention. Global dashboards receive only approved aggregates.

If the approved regional model is unavailable, policy may queue the request or allow a separately evaluated approved route. It must not fall back to a public endpoint to preserve availability.

Clinical decisions remain outside the gateway. A model refusal, summary, or extraction is evidence for an authorized workflow, not a diagnosis or care directive.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Central gateway | Consistent policy and evidence | Shared latency and blast radius |
| SDK in every service | Local flexibility | Policy drift and key sprawl |
| Canonical API | Easier migration | Semantic loss |
| Provider-specific API | Full capability | Lock-in and duplicated controls |
| Capability routing | Intent-based selection | Registry and testing burden |
| Fixed route | Predictability | Weak resilience |
| Dynamic routing | Capacity/cost optimization | Behavior variability |
| Evaluated fallback | Better availability | Ongoing evaluation burden |
| No fallback | Stable semantics | More visible outages |
| Buffered response | Full-output validation | Higher first-token latency |
| Direct streaming | Lower perceived latency | Partial-output safety complexity |
| Regional data planes | Residency and resilience | Operational duplication |
| Synchronous global quota | Strong limit | Cross-region latency/failure |
| Regional budget leases | Partition tolerance | Bounded temporary overspend |
| Prompt logging | Easier debugging | Privacy and retention risk |
| Metadata/digests | Lower exposure | Harder content debugging |
| Semantic cache | Cost/latency savings | Incorrect or cross-scope reuse |
| Hedging | Lower tail latency | Double cost and disclosure |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Gateway becomes universal business layer | Coupling and unclear ownership | Keep policy/transport boundary narrow |
| Lowest-common-denominator API | Lost capabilities without true portability | Capability registry and extensions |
| Provider feature assumed equivalent | Silent behavioral error | Contract tests and route evaluations |
| Floating alias changes unnoticed | Quality or safety drift | Observed versions and canaries |
| Soft score overrides residency | Policy breach | Hard eligibility filter first |
| Tenant derived from prompt | Spoofing and data leak | Trusted workload identity |
| One provider key for all apps | Large blast radius | Scoped accounts and rotation |
| Raw prompts logged globally | Confidential-data exposure | Regional, redacted, purpose-bound evidence |
| Retry after partial stream | Duplicate/contradictory output | Explicit partial failure |
| Retry storm during throttling | Provider and gateway collapse | Budgets, backoff, breaker |
| Hedge sends restricted data twice | Contract/privacy breach | Data-class eligibility |
| Fallback changes tool semantics | Bad action proposal | Pre-evaluated route chains |
| Gateway executes tool calls | Uncontrolled side effect | Durable workflow boundary |
| Slow client fills memory | Gateway outage | Bounded buffers and cancellation |
| Control plane on request path | Global outage | Immutable local bundles |
| Stale policy never expires | Revoked route remains active | Bundle validity and emergency deny |
| Split-brain route configuration | Inconsistent behavior | Revisioned signed bundles |
| Cross-region failover ignores residency | Regulatory breach | Tenant home-region policy |
| Partition defeats global budget | Unbounded spend | Regional leases |
| Transport health hides quality failure | Harmful successful responses | Semantic canaries and outcome metrics |
| Cache omits authorization/version | Cross-tenant or stale response | Complete scoped key or no cache |
| Token estimate treated as invoice | Incorrect chargeback | Usage settlement and billing reconciliation |
| Provider timeout outcome ignored | Hidden cost/duplicate attempt | Attempt lineage and reconciliation |

## What to measure

Measure by tenant, application, workload, route, region, release, and fallback state:

- gateway availability and added latency;
- admission rejection and queue delay;
- time to first token and completion;
- stream cancellation and partial-output rate;
- provider errors, throttling, retries, hedges, and circuit state;
- eligible-route count and no-compliant-route rate;
- fallback frequency and quality delta;
- tool/schema validity;
- task quality and safety by route;
- input/output/cached tokens;
- estimated versus settled cost;
- budget-lease utilization and reconciliation error;
- provider alias/revision drift;
- policy/configuration propagation and stale-bundle age;
- cross-region failover attempts and policy denials;
- unauthorized route attempts;
- evidence completeness and sensitive-log findings;
- downstream approval, correction, and incident outcomes.

The gateway is successful when applications gain controlled portability without losing semantic honesty.

## Interview prompt

> Design a multi-tenant AI gateway for construction and healthcare operations. It must support several hosted and private models, streaming, tools, structured output, quotas, cost attribution, data residency, evaluated fallback, and multi-region continuity. Provider APIs and model behavior differ, and the control plane may be unavailable during an incident.

Cover the request contract, capability registry, routing constraints, adapters, credentials, quotas, streaming, retries, fallback, caching, regional architecture, control-plane publication, observability, security, evaluation, and failure recovery.

## Worked answer

**One-line conclusion:** Put a thin regional gateway on the inference path and publish immutable policy bundles from a separate control plane, routing only among pre-evaluated, policy-eligible model releases while exposing every degradation and fallback.

“I would define a versioned capability request rather than exposing provider model names. Trusted workload identity supplies tenant and purpose. The envelope declares modality, tools, structured schema, deadline, data class, residency, risk, and budget.

The gateway authenticates, authorizes, reserves capacity, and loads one immutable configuration revision. It first filters routes by hard constraints—region, data contract, capability, context, evaluation gate, and quota—then optimizes quality, health, latency, and cost. Provider adapters preserve differences and return explicit unavailable fields rather than inventing equivalence.

Streaming uses bounded buffers, cancellation, total and idle deadlines, and concurrent-stream quotas. I never retry invisibly after output reaches the user. Retries have a small deadline-bound budget; hedging and fallback require data-policy approval and pre-release evaluation. High-risk work queues or abstains when no equivalent route exists.

Applications authenticate to the gateway; regional secret stores hold scoped provider credentials. Tool calls return to a durable application workflow for validation, approval, idempotency, and external-outcome reconciliation.

Regional data planes keep prompts in approved locations. A global control plane publishes signed, versioned region-specific bundles; the data plane keeps the last compatible good bundle. Tenant home-region and data policy gate failover. Global budgets allocate expiring regional leases so a partition cannot create unlimited spend.

I trace every provider attempt and record route/config/model/adapter versions, latency, usage, fallback, schema validity, and privacy-safe evidence. Semantic canaries detect quality degradation even when transport is healthy. Each route chain is evaluated by workload and slice before canary promotion.

For healthcare PHI, no approved regional route means queue or fail closed—not an unapproved public fallback. Construction ERP proposals and healthcare operational outputs remain approval-bound outside the gateway.”

## Practice lab

Design the gateway when:

- three providers expose different tool and structured-output semantics;
- a construction tenant requires US-only processing;
- a healthcare tenant sends PHI only to one approved regional account;
- a provider silently changes a floating model alias;
- an interactive stream emits 200 tokens and then fails;
- a slow client stops reading;
- one provider returns throttles for ten minutes;
- month-end batch jobs consume most token capacity;
- a fallback model is cheaper but fails handwritten invoices;
- global control-plane connectivity is lost;
- two regions disagree on a policy revision;
- the primary region fails;
- global cost tracking is unavailable;
- a model proposes an ERP action;
- provider billing differs from gateway token estimates.

Specify the canonical contract, hard eligibility rules, capability registry, adapter tests, admission and regional budget leases, streaming behavior, retry/fallback policy, credential isolation, control-plane publication, regional failover, observability, evaluation gates, and action boundary.

Then adapt the design so PHI never crosses an unauthorized region, raw prompts are not globally logged, and the system queues rather than silently weakening safety.

## References

- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [OpenTelemetry semantic conventions for generative AI systems](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [NIST SP 800-207 — Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [IETF RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [IETF RFC 6585 — Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
