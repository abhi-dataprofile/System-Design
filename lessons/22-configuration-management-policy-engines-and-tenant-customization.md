# 22 — Configuration management, policy engines, and tenant customization

Previous: [Privacy engineering, retention, and deletion workflows](21-privacy-engineering-retention-and-deletion-workflows.md). Study time: 50–60 minutes including the exercise.

## Mental model

Configuration is executable behavior. A threshold, workflow toggle, model choice, approval rule, or integration mapping can change correctness just as surely as a code deployment.

Model the decision path as:

```text
typed inputs + trusted context + versioned configuration + policy
                        ↓
             deterministic decision
                        ↓
       recorded explanation + immutable snapshot
```

Separate four concerns:

- **code** defines capabilities and invariants;
- **configuration** selects bounded operating values;
- **policy** decides whether an action is allowed or required;
- **feature rollout** controls who receives a compatible behavior version.

> Centralize configuration ownership, evaluate policy from trusted context, and bind every consequential decision to the exact versions that produced it.

Tenant customization should compose supported primitives. It should not become one code branch, database column, or prompt fork per customer.

## When to use each mechanism

Use configuration for values expected to vary safely without rebuilding software:

- per-tenant limits, timeouts, routing destinations, locale, and notification preferences;
- connector endpoints and vendor field mappings;
- model or retrieval profiles chosen from an approved catalog;
- review thresholds within enforced bounds;
- retention schedules selected from approved policy classes.

Use a policy engine when the result depends on identities, relationships, attributes, resource state, or regulatory/business rules:

- whether a user may read a project;
- whether an invoice needs one or two approvals;
- whether a clinical action requires a licensed role;
- whether data may leave a region;
- whether a proposed tool call is allowed now.

Use feature flags for temporary release control, experiments, emergency disablement, and progressive exposure. A flag is not a permanent configuration database.

Keep immutable safety constraints in code or a separately governed policy layer. Do not let a tenant setting disable tenant isolation, audit, encryption, required clinical review, or idempotency.

## Configuration hierarchy and precedence

A useful hierarchy may include:

```text
platform default
  → environment
    → region
      → tenant
        → project/facility
          → workflow
            → user preference
```

More layers increase flexibility and debugging cost. Define precedence once and expose the resolved result.

For every key, specify:

- stable name and semantic description;
- type, units, allowed range, and default;
- which scopes may override it;
- whether inheritance is allowed;
- sensitivity classification;
- change owner and required approval;
- compatibility and activation constraints;
- whether it affects in-flight work;
- deprecation and migration behavior.

“Missing,” “inherit,” “explicit null,” “false,” and zero are distinct states. Avoid truthiness-based resolution.

A resolver should return not only the value but its provenance:

```json
{
  "key": "invoice.approval.threshold_minor_units",
  "value": 10000000,
  "source_scope": "tenant/acme",
  "config_version": "cfg_1842",
  "schema_version": 3,
  "evaluated_at": "2026-10-15T12:00:00Z"
}
```

## Control plane and data plane

Separate configuration management from request serving.

The **control plane** handles authoring, validation, approval, versioning, rollout, rollback, audit, and distribution.

The **data plane** reads an immutable, validated snapshot and serves decisions without depending on an editor UI or a slow remote database on every request.

A safe flow:

1. user proposes a typed change;
2. server authorizes the target scope and key;
3. schema and cross-field invariants validate it;
4. policy simulation shows affected workflows;
5. required reviewers approve;
6. a new immutable version is published;
7. distribution sends the version to subscribers;
8. consumers acknowledge and activate atomically;
9. monitoring checks expected and unexpected effects;
10. rollback republishes a prior compatible version.

Draft, approved, published, active, superseded, and revoked should be explicit states.

## Typed schemas and invariants

Store configuration as typed documents or rows validated against a versioned schema. Reject unknown keys unless forward compatibility explicitly permits them.

Validation has multiple levels:

- **field:** type, range, format, enum;
- **object:** cross-field rules;
- **referential:** model, queue, role, region, and connector exist;
- **capability:** target service supports the schema/version;
- **safety:** value stays within platform-enforced bounds;
- **semantic:** unit, timezone, currency, and lifecycle meaning are explicit.

Examples:

- retry maximum must fit within the workflow deadline;
- approval threshold must use the tenant’s configured currency;
- a model profile handling PHI must be approved for the region and data class;
- enabling automatic execution requires idempotent connector support;
- retention duration cannot undercut the governing record class.

Do not accept arbitrary code, SQL, regular expressions without limits, or user-supplied templates with unrestricted execution.

## Versioning and snapshots

Mutable “current settings” are insufficient for durable workflows. Store:

- configuration document ID and immutable version;
- schema version;
- policy bundle version;
- feature-flag evaluation version;
- effective interval;
- author, approver, reason, and ticket;
- content hash;
- compatibility metadata.

At workflow start, either pin a snapshot or define which fields may be reevaluated. Safety and authorization should normally be rechecked at execution; cosmetic display preferences may follow current values.

For a six-hour invoice workflow, record the snapshot used to extract, route, and propose. If an approval threshold changes before execution, compare old and new policy. A stricter rule may invalidate approval; a presentation-color change should not.

Snapshot references are better than copying secrets or entire documents into every event.

## Distribution and consistency

Configuration distribution is a consistency problem.

Options:

| Pattern | Benefit | Cost or risk |
| --- | --- | --- |
| Read central store per request | Fresh and simple authority | Latency and outage coupling |
| Poll versioned snapshots | Bounded dependency | Staleness window |
| Push/watch stream | Fast propagation | Disconnect and ordering complexity |
| Local cache with lease | Resilient reads | Expiry behavior must be defined |
| Embedded config in deploy | Reproducible | Slow tenant changes |

Use monotonically ordered versions or generations. Consumers reject older versions and activate a complete bundle atomically rather than mixing related keys.

For high-risk changes, require quorum or explicit acknowledgements from critical consumers before activation. For ordinary preferences, eventual convergence may be acceptable.

Define maximum staleness and behavior on expiry:

- continue with last-known-good for low-risk display settings;
- fail closed for authorization or data-residency policy;
- enter a restricted mode for action execution;
- never silently fall back to permissive defaults.

## Policy evaluation

A policy request should contain trusted, typed facts:

- authenticated subject and roles;
- tenant and resource identity;
- action and normalized parameters;
- resource version and classification;
- relationship facts such as project membership;
- environment, region, time, and risk signals;
- approval and workflow state.

The application—not the model or browser—builds authoritative context. Treat model output as an untrusted proposal.

A policy response should include:

```json
{
  "decision": "require_two_approvals",
  "policy_version": "pol_392",
  "reason_codes": ["AMOUNT_OVER_LIMIT", "NEW_VENDOR"],
  "obligations": {
    "roles": ["project_manager", "controller"],
    "expires_in_seconds": 3600
  }
}
```

Keep evaluation deterministic for the same input and bundle. Make reason codes stable enough for audit and user interfaces, but do not expose sensitive policy internals that enable abuse.

## Authorization is not only a boolean

Useful policy results include obligations:

- redact specified fields;
- restrict query scope;
- require a purpose or ticket;
- require step-up authentication;
- require one or more approver roles;
- cap amount, row count, or export size;
- select an approved region or model;
- require logging, watermarking, or expiry;
- deny automatic execution but allow a draft.

Enforce obligations in the component performing the action. A UI hiding a button is not authorization.

## Secrets and sensitive configuration

Do not store credentials in ordinary configuration documents or feature-flag payloads. Store references to a secrets manager and resolve them through workload identity at use time.

Separate permissions to:

- view configuration structure;
- view sensitive values;
- edit drafts;
- approve/publish;
- rotate secrets;
- read audit history.

Never return secrets to browsers merely because a user may edit the integration. Prefer write-only secret fields and connection tests executed by a trusted service.

Secret rotation should create a new version, support overlapping validity when possible, test it, switch consumers, and revoke the old credential. Audit references the secret version, not its value.

## Safe customization boundaries

Offer a product vocabulary, not arbitrary implementation access:

- approved workflow steps;
- typed conditions;
- bounded thresholds;
- role and approval templates;
- vendor mapping functions;
- notification templates with escaped variables;
- approved model/retrieval profiles;
- webhooks with signed, scoped events.

When customization becomes sufficiently complex, compile it into an intermediate representation and validate that representation before execution. Bound nesting, iterations, expression cost, payload size, external calls, and total runtime.

Avoid customer-specific forks. If one tenant needs a reusable capability, introduce it behind a versioned interface and decide whether it belongs in the supported product.

## Feature flags and experiments

Flag evaluations need:

- flag key and version;
- subject/tenant attributes from trusted context;
- deterministic bucketing key;
- variant;
- reason;
- evaluation time;
- default behavior.

Use stable hashing so users do not switch cohorts across requests. For multi-tenant enterprise workflows, bucket by tenant or project when mixed behavior inside one organization would be confusing or unsafe.

A kill switch should disable a narrowly scoped risky capability without disabling read-only recovery and reconciliation. Test kill switches before incidents.

Remove expired flags. Permanent nested flags multiply system states and make testing incomplete.

Never use an experiment to bypass required consent, authorization, clinical review, or other mandatory controls.

## Safe rollout and rollback

Treat configuration and policy releases like software:

1. static validation and linting;
2. unit tests for allow, deny, boundary, and missing-data cases;
3. replay against sanitized historical decisions;
4. policy/config diff with affected scopes;
5. shadow evaluation without enforcement;
6. canary tenants or internal users;
7. monitored progressive rollout;
8. rollback or kill switch;
9. post-release decision comparison.

Test both expected decisions and invariants such as “no cross-tenant read is ever allowed.”

Rollback is not always safe. If a new policy already created approvals or external actions, restoring the old bundle cannot erase those effects. Reconcile state and decide how in-flight workflows handle the transition.

## Caching policy decisions

Cache only when all decision inputs and versions are represented in the key:

```text
subject + tenant + resource/version + action + context
+ policy bundle + relevant config generation
```

Use short TTLs for volatile membership or resource state. Invalidate on role revocation, project membership change, resource reclassification, policy publication, or emergency deny.

Do not cache a broad “user is allowed” result and reuse it for different resources or actions. Deny caching may also be unsafe when emergency access or newly granted duties must take effect quickly.

For critical execution, reevaluate immediately before the side effect.

## Testing the state space

Configuration creates combinatorial behavior. Control it through:

- pairwise or property-based testing across key combinations;
- golden tests for important tenants and workflow templates;
- invariant tests independent of configuration;
- schema migration tests;
- old consumer/new bundle compatibility tests;
- fuzzing invalid and boundary values;
- replay of historical policy requests;
- differential tests between old and new versions;
- chaos tests for stale, unavailable, reordered, or partially distributed bundles.

Maintain a small set of supported profiles rather than allowing every key to vary independently.

## Construction example

A contractor customizes invoice approval:

- platform policy requires tenant isolation, audit, and idempotent ERP writes;
- tenant configuration sets USD and a $100,000 controller threshold;
- one project adds a project-manager review above $25,000;
- a feature flag enables a new extraction model for two pilot projects;
- vendor mappings translate cost codes to the ERP.

An invoice for $120,000 starts under configuration `cfg_1842` and policy `pol_392`. The decision requires project-manager and controller approval. The case records the resolved threshold provenance, mapping version, model profile, policy reasons, and plan hash.

Before execution, the tenant lowers the controller threshold and the project removes an approver. The worker reevaluates current authorization and safety rules. It does not blindly use the old approval. If required roles or material action parameters changed, it marks the approval stale. The ERP request keeps the same stable operation identity only when the approved action itself is unchanged.

A bad cost-code mapping is rolled back to its last-known-good version. Previously posted invoices enter reconciliation; rollback does not pretend those external effects vanished.

## Healthcare operations variation

A healthcare workflow supports facility-level routing and clinical-role policy. Platform invariants require patient-scoped authorization, minimum necessary data, approved regional processing, and clinical review for medication changes.

A facility may configure notification windows and choose among approved scheduling workflows. It cannot configure away licensed-role requirements or route PHI to an unapproved model endpoint.

Policy context uses authenticated workforce identity, facility relationship, patient/encounter scope, action, current record version, and emergency-access state. Break-glass access is a distinct policy path with reason capture, elevated audit, narrow duration, and retrospective review—not a permissive default.

If the policy service or its lease expires, the system may preserve low-risk read-only operations under a last-known-good bundle but holds consequential clinical actions until authoritative policy is available.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Central control plane | Governance and audit | Availability dependency |
| Local snapshots | Fast resilient evaluation | Bounded staleness |
| Deep override hierarchy | Flexible customization | Hard provenance/debugging |
| Few supported profiles | Testable behavior | Less tenant flexibility |
| Policy as code | Reviewable and repeatable | Specialist skills and rollout risk |
| Rules in application code | Strong typing | Slow policy changes |
| Feature flags | Progressive delivery | State-space and cleanup burden |
| Pin workflow snapshot | Reproducibility | May miss new safety rule |
| Always use latest | Immediate change | In-flight inconsistency |
| Last-known-good | Outage resilience | Stale-policy exposure |
| Fail closed | Protects consequential actions | Availability loss |
| Arbitrary tenant scripts | Maximum flexibility | Security and support risk |

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| Untyped key/value store | Invalid behavior reaches production | Versioned schema and invariants |
| Precedence is implicit | Tenant sees surprising value | One resolver with provenance |
| Missing value becomes false/zero | Safety rule disabled | Explicit missing/inherit/null semantics |
| Partial bundle activation | Contradictory rules | Atomic versioned snapshot |
| Old update arrives late | Configuration rollback by accident | Monotonic generation check |
| Permissive outage default | Unauthorized action | Risk-based fail closed/restricted mode |
| Current config overwrites history | Decisions cannot be explained | Immutable versions and hashes |
| Workflow pins old authorization | Revoked user still acts | Reevaluate at execution |
| Flag becomes permanent policy | Poor governance and testing | Promote or delete with owner/expiry |
| Tenant can override invariant | Isolation or compliance failure | Non-overridable platform constraints |
| Secret stored in flag | Credential exposure | Secret reference and workload identity |
| Arbitrary template executes code | Injection or resource exhaustion | Sandboxed typed DSL and limits |
| Cached decision omits resource | Privilege spreads across objects | Complete versioned cache key |
| Rollback ignores prior writes | External state remains wrong | Reconciliation plan |
| Customer-specific code fork | Upgrade and reliability debt | Supported composable primitives |
| Model supplies policy context | Authorization manipulation | Server-built trusted facts |

## What to measure

Track configuration publishes, validation failures, approvals, rollback frequency, stale-consumer count, propagation latency, active version by service, last-known-good age, policy evaluation latency and availability, allow/deny/obligation rates, decision changes between versions, cache hit and invalidation rates, emergency access, overrides by scope, expired flags, tenant-specific code paths, stale workflow approvals, secret-rotation status, and incidents attributable to configuration.

Alert on partial activation, unknown schema versions, permissive fallback, widening authorization, sudden decision-rate changes, and bundles that critical consumers have not acknowledged.

## Interview prompt

> Design configuration management and policy evaluation for a multi-tenant AI operations platform. Tenants customize construction invoice workflows and healthcare facilities customize operational routing. The platform uses feature flags, model profiles, external connectors, long-running approvals, and high-consequence tool actions.

Cover configuration hierarchy, typed schemas, control/data planes, distribution consistency, versioning, policy inputs and obligations, tenant safety boundaries, secrets, caching, rollouts, rollback, long-running workflows, observability, and outage behavior.

## Worked answer

**One-line conclusion:** Make customization a versioned, typed control plane whose immutable snapshots and deterministic policies select bounded behavior without weakening platform invariants.

“I would define an explicit hierarchy from platform to tenant and project, with a central resolver returning value plus source and version. Each key has type, units, allowed scopes, bounds, owner, and compatibility rules. A control plane authorizes, validates, simulates, approves, publishes, and audits immutable bundles; data-plane services consume atomically activated snapshots and reject older generations.

Policy evaluation receives authenticated server-built subject, tenant, resource/version, action, environment, and risk facts. It returns a decision, stable reason codes, obligations, and policy version. Platform invariants—tenant isolation, required clinical roles, audit, and idempotency—cannot be overridden. Secrets remain in a secrets manager. Decision caches include all relevant identities, resource versions, policy bundle, and config generation, while consequential actions reevaluate immediately before execution.

Long workflows record their starting snapshot but recheck current authorization and stricter safety rules. Releases run static tests, historical replay, shadow comparison, canary rollout, and monitored expansion. Consumers use last-known-good only within a lease; expired authorization policy fails closed or enters restricted mode. Rollback restores behavior but separately reconciles external effects already produced. Construction tenants compose approval thresholds and ERP mappings; healthcare facilities configure operations only inside patient, role, region, and clinical-safety boundaries.”

## Practice lab

Design the control plane for this scenario:

- a tenant sets a $100,000 invoice threshold;
- a project overrides it to $25,000;
- a pilot flag selects a new extraction model;
- a cost-code mapping changes during a six-hour approval;
- a user’s project role is revoked before execution;
- one service receives configuration version 43 while another remains on 42;
- the policy service becomes unavailable;
- rollback follows several possibly accepted ERP writes;
- a healthcare facility wants to disable clinical review for speed.

Specify hierarchy and precedence, schemas and invariants, bundle versioning, distribution and activation, policy request/response, snapshot versus reevaluation rules, cache keys, outage behavior, rollout tests, rollback/reconciliation, and observability.

## References

- [Open Policy Agent documentation](https://www.openpolicyagent.org/docs/latest/)
- [CNCF — OpenFeature specification](https://openfeature.dev/specification/)
- [NIST SP 800-53 Rev. 5 — Configuration Management controls](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NIST SP 800-162 — Attribute Based Access Control](https://csrc.nist.gov/pubs/sp/800/162/upd2/final)
- [Google SRE Book — Configuration Design and Best Practices](https://sre.google/sre-book/configuration-design/)
