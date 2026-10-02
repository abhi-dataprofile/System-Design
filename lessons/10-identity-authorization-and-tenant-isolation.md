# 10 — Identity, authorization, and tenant isolation

Previous: [Replication, consistency, and failover](09-replication-consistency-and-failover.md). Study time: 45–55 minutes including the exercise.

## Mental model

Security decisions answer different questions:

- **Authentication:** Who or what is making the request?
- **Authorization:** May this principal perform this action on this resource now?
- **Tenant isolation:** Can any path cause one customer’s data or capacity to cross into another’s boundary?
- **Audit:** Can we later reconstruct the decision and consequential action?

A useful authorization tuple is:

```text
(subject, tenant, action, resource, context) -> allow or deny
```

Context can include membership status, project assignment, resource state, time, network/device assurance, purpose of use, and whether the action requires approval.

Identity is an input to authorization, not the final decision. A valid login does not imply access to every tenant, project, patient, document, model, or tool.

The governing rule is:

> **Derive tenant and permission from trusted server-side state, deny by default, and enforce the decision at every data and side-effect boundary.**

## When to use each authorization model

| Model | Good fit | Main tradeoff |
| --- | --- | --- |
| Role-based access control (RBAC) | Stable job functions such as viewer, project manager, approver | Role explosion and coarse exceptions |
| Attribute-based access control (ABAC) | Rules involving tenant, project, region, sensitivity, time, or resource state | Harder policy testing and debugging |
| Relationship-based access control (ReBAC) | Graph relationships such as member-of, assigned-to, owns, treating-clinician | Relationship freshness and graph complexity |
| Access-control lists (ACLs) | Small resource-specific grants | Difficult administration at scale |
| Capability/delegation token | Narrow temporary permission for one operation | Leakage/replay risk and revocation complexity |

Production systems often combine them. RBAC supplies a baseline, relationships scope it to the resource, and attributes add contextual constraints.

Keep business roles separate from infrastructure roles. “Project approver” should not translate directly into a cloud administrator role.

## Authentication flow

For human users, use an established identity provider and standard protocol such as OpenID Connect. The application should:

1. redirect the user to the trusted identity provider;
2. receive an authorization response through a protected redirect flow;
3. validate the returned tokens;
4. map the external subject to an internal principal;
5. resolve current tenant memberships and application permissions;
6. create a bounded session.

Do not collect or store user passwords when federation fits.

### Validate tokens completely

For a signed token, validate at least:

- signature using the expected algorithm and trusted key;
- issuer;
- audience;
- expiration and not-before time with bounded clock skew;
- token type or intended use;
- authorization context required by the endpoint;
- nonce/state/PKCE where the flow requires them.

Do not accept a token merely because its signature is valid. A token issued for another service or environment can be correctly signed and still be invalid for this API.

Keys rotate. Cache trusted key sets with bounded freshness, handle unknown key IDs safely, and do not disable verification during provider trouble.

Access tokens authorize API calls. ID tokens describe an authentication event for the client; they should not be casually substituted for access tokens.

## Sessions, revocation, and freshness

Short-lived sessions reduce exposure but do not eliminate revocation needs.

Possible designs:

| Session | Strength | Tradeoff |
| --- | --- | --- |
| Signed self-contained token | Fast validation without central lookup | Permission changes remain stale until expiry unless checked |
| Opaque server session | Immediate server-side control and revocation | Shared state dependency |
| Short token plus current membership lookup | Bounded identity token with fresher authorization | More calls/cache design |

For sensitive actions, re-evaluate current membership, resource state, and permission even if the token remains valid. A user removed from a project should not retain approval rights until a long-lived token expires.

Use a policy or membership version to invalidate cached decisions. Fail closed if current authorization cannot be established for protected data or consequential actions.

Session cookies should be secure, HTTP-only, same-site as appropriate, rotated after privilege changes, and protected from cross-site request forgery when the browser automatically attaches them.

## Authorization policy design

State policy in business terms:

```text
allow invoice.approve when
  subject is active
  and subject belongs to resource.tenant
  and subject is assigned to resource.project
  and subject has role project_approver
  and invoice.status == PENDING
  and invoice.amount <= subject.approval_limit
  and separation_of_duties permits it
```

Keep policy evaluation deterministic and testable. Return a stable reason code internally, but avoid disclosing resource existence or sensitive policy details to unauthorized clients.

Distinguish:

- **policy decision point:** evaluates the rule;
- **policy enforcement point:** blocks or permits the operation;
- **policy information point:** supplies trusted identity, relationship, and resource attributes.

Central policy logic improves consistency, but enforcement must remain at each service and data boundary. A gateway-only authorization check is insufficient when background workers, internal APIs, and tools can reach data directly.

## Tenant resolution

Never trust a raw `X-Tenant-ID`, URL parameter, model output, or request body as proof of tenant membership.

A safe flow:

1. validate the principal;
2. load active memberships from trusted state;
3. resolve the requested tenant against those memberships;
4. create an internal request context;
5. pass that context through typed interfaces;
6. include tenant predicates in every database, cache, search, storage, and queue operation.

A user may belong to multiple tenants. The product should make the active tenant explicit and auditable, but the server still validates every resource against it.

Random IDs make guessing harder; they do not provide authorization.

## Tenant isolation across the data path

Isolation is an end-to-end property.

### Relational database

Use tenant-scoped keys and constraints:

```sql
PRIMARY KEY (tenant_id, invoice_id)

FOREIGN KEY (tenant_id, project_id)
  REFERENCES projects(tenant_id, project_id)
```

Queries include `tenant_id` explicitly. Database row-level security can add defense in depth when session context is set correctly and connection pooling cannot leak it. Test the database owner/bypass roles and maintenance paths; some roles can bypass row policies.

Separate schemas or databases can strengthen blast-radius and residency isolation, but increase migrations, connection management, analytics, and cost. Choose by risk and scale rather than assuming one topology fits every tenant.

### Cache

Keys include tenant, authorization-relevant versions, and representation versions. Cached values must not outlive a revocation policy for sensitive access.

### Object storage

Clients receive short-lived URLs only after server-side authorization to one exact immutable object version. Object keys are opaque and do not contain PHI, secrets, or descriptive customer data.

### Search and vector indexes

Apply tenant/project/patient authorization inside retrieval before candidates, snippets, counts, or embeddings reach the model. Post-filtering a small top-`k` can leak and can destroy recall.

### Queues and events

Messages carry a validated tenant identifier and resource IDs, not ambient user trust. Consumers re-establish service authorization and validate resource ownership. Dead-letter queues, replay tools, and logs require the same isolation as the main path.

### Analytics and logs

Do not place sensitive data in labels, traces, prompts, or error messages. Operational staff access is itself authorized and audited. Aggregations need minimum-group or de-identification rules where small counts can reveal individuals.

## Service identity and the confused deputy

Every service and workload needs its own identity. Prefer short-lived workload credentials issued by the platform over shared static secrets.

The confused-deputy problem occurs when a privileged service is tricked into using its authority for an unauthorized caller. Prevent it by carrying the caller’s validated context and checking both:

- may the calling service invoke this operation?
- may the originating subject perform it on this resource?

Use delegation with the narrowest needed scope, audience, tenant, resource, action, and expiration. Do not forward a broad user token to arbitrary downstream tools.

For background work, persist an immutable authorization snapshot or operation authorization record when the job is accepted, plus the policy governing whether permissions must be rechecked at execution. A two-hour document export should not rely on an expired browser session, but it also should not continue after a policy that requires revocation.

## Secrets and machine credentials

Store credentials in a managed secret system, not source code, images, prompts, tickets, or ordinary environment dumps.

Design for:

- least-privilege scopes;
- separate identities by environment and service;
- automatic rotation;
- short lifetimes;
- audit of secret reads;
- safe failure during rotation;
- revocation playbooks;
- no secret values in logs or model context.

Prefer identity federation to long-lived access keys. Encryption protects stored bytes; authorization controls who can request decryption. Key-management permissions must be separated from ordinary data access where risk requires it.

## AI agents and tool authorization

A model is not a principal you should trust with ambient authority. It proposes; deterministic code authorizes and executes.

For every tool call:

1. parse the requested tool and typed arguments;
2. bind the original authenticated user, active tenant, and operation;
3. check the user’s permission for that tool and resource;
4. validate state, limits, and approval requirements;
5. show a human-readable preview for consequential actions;
6. use an idempotency key;
7. execute with a scoped service identity;
8. record the outcome and audit event.

Retrieved documents and user messages are untrusted content. A prompt injection cannot grant a new permission, change tenant context, expose secrets, or bypass a human approval gate.

Use separate read and write tools. Default the agent to read-only. A broad “execute SQL” or “call any URL” tool defeats meaningful authorization. Allowlists, schemas, network egress policy, and resource-level checks narrow the blast radius.

Memory is another data store. Conversation summaries, embeddings, checkpoints, traces, and evaluation examples inherit tenant isolation, retention, and deletion requirements.

## Human approval and separation of duties

High-impact actions can require:

- requester and approver to be different people;
- two approvers above an amount threshold;
- stronger authentication or recent re-authentication;
- reason/purpose-of-use;
- a preview of exact resources and changes;
- approval expiration;
- reauthorization if the underlying resource changes.

Approval applies to a specific immutable action plan. If invoice amount, destination account, document version, or tool arguments change, invalidate the approval.

The executor must re-check authorization and resource version at execution time. Approval does not guarantee the action is still permitted hours later.

## Break-glass access

Emergency access is a designed exception, not a shared administrator password.

A break-glass flow should have:

- narrow eligible personnel;
- strong authentication;
- explicit reason and incident/ticket;
- time-limited elevation;
- resource and action limits;
- prominent warning;
- immutable high-priority audit;
- real-time or prompt review;
- automatic expiry;
- post-event attestation.

In healthcare, emergency patient access may be necessary, but it should not become a routine workaround for poor role design.

## Audit trail

Security logs should answer:

- who acted and through which authenticated session/workload;
- active tenant and delegated subject;
- action and exact resource/version;
- policy decision and stable reason;
- request/operation/trace ID;
- before/after state references for consequential changes;
- approval identity and plan version;
- tool/model/prompt policy version where AI participated;
- result, external reference, and uncertain outcome;
- timestamp from trusted infrastructure.

Audit logs should be append-oriented, access-controlled, retained according to policy, and monitored for gaps. Avoid storing secrets or unnecessary raw PHI in the audit event.

Application logs help debug. Audit records prove security-relevant events. They have different completeness, retention, and access requirements.

## Construction example

A subcontractor, project manager, controller, and auditor use the same multi-tenant assistant.

- The identity provider authenticates users; the app maps each external subject to internal memberships.
- The subcontractor can upload documents only to assigned projects.
- The project manager can read project records and propose invoice approval.
- The controller can approve within a tenant-defined amount limit but cannot approve an invoice they submitted.
- The auditor has read-only access to finalized records and audit history.
- Database queries and constraints include tenant/project scope.
- Search pre-filters every chunk by current membership.
- Signed object URLs are issued only for an authorized immutable version.
- The agent can draft an approval but cannot execute it until deterministic policy and human approval succeed.
- The ERP connector receives one scoped operation with a stable idempotency key.
- Membership revocation increments a policy version, invalidates sensitive caches, and blocks later tool calls.

A valid request for invoice `42` in Tenant A must not discover whether Tenant B also has invoice `42`.

## Healthcare variation

Use established healthcare authorization context where available, such as SMART on FHIR scopes, but do not mistake a broad scope for complete patient-level authorization.

A clinical decision can depend on:

- practitioner identity and current employment;
- treatment relationship and care team;
- organization and department;
- patient/encounter context;
- resource type and sensitivity;
- purpose of use;
- consent and legal restrictions;
- emergency break-glass state.

Validate SMART/OAuth issuer and audience, authorize the requested FHIR resource on every call, and keep vendor/application identity in the audit trail. A clinician who can read one patient’s observations must not search embeddings across all patients and filter afterward.

PHI can appear in prompts, traces, caches, vector indexes, evaluation datasets, support exports, and DLQs. Each path needs minimum-necessary access, retention, deletion, encryption, and auditing.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Short-lived tokens | Limits stolen-token lifetime | Refresh complexity and more identity dependency |
| Current policy lookup per request | Fresh revocation | Latency and availability dependency |
| Cached authorization | Lower latency | Stale grants; needs version/invalidation |
| Shared database with tenant keys | Operational efficiency | Larger blast radius if a predicate is missed |
| Database per tenant | Stronger isolation and easier tenant restore | Cost and fleet-management complexity |
| Central policy service | Consistent decisions | Critical dependency and latency |
| Embedded library plus versioned policy | Low latency | Coordinated rollout and drift risk |
| Fine-grained ABAC/ReBAC | Precise least privilege | Policy complexity and test burden |
| Human approval | Reduces consequential error | Friction and slower operations |
| Break-glass access | Emergency availability | Insider risk and review burden |

Security architecture balances risk, usability, latency, operational capacity, and regulatory duties. “Zero trust” does not mean zero availability; it means no implicit trust based only on network location.

## Failure modes

| Failure | Consequence | Response |
| --- | --- | --- |
| API trusts client tenant header | Cross-tenant access | Resolve membership server-side |
| Signature checked but audience ignored | Token for another API accepted | Full issuer/audience/type validation |
| Gateway is only enforcement point | Worker/internal path bypasses policy | Enforce at service and data boundary |
| Cache key omits tenant/policy version | Data survives tenant or permission change | Scoped key and explicit invalidation |
| Search filters after top-k | Leakage or empty authorized results | Pre-filter inside retrieval |
| Connection pool leaks DB tenant context | Next request inherits wrong tenant | Transaction-local context, reset, tests |
| Long job keeps revoked authority | Export continues after removal | Defined authorization snapshot/recheck policy |
| Service uses broad static key | Large compromise blast radius | Short-lived workload identity and least privilege |
| Agent follows prompt injection | Unauthorized tool or data access | Deterministic tool policy and scoped executor |
| Approval plan mutates afterward | User approves one action; system executes another | Hash/version the exact plan and reapprove changes |
| Break-glass becomes routine | Persistent over-privilege | Time limits, alerts, review, role repair |
| Audit logs contain raw secrets/PHI | Secondary data breach | Structured minimum-necessary events |
| Authorization outage fails open | Protected data exposed | Fail closed; preserve safe public/status paths |

## What to measure

Track security outcomes, not just login success:

- authentication successes/failures and suspicious token validation errors;
- authorization allows/denies by stable reason and action;
- cross-tenant security tests and policy regression results;
- membership-to-enforcement propagation time;
- cache/search entries using stale policy versions;
- privileged and break-glass sessions, duration, and review completion;
- service credentials by age, scope, and rotation status;
- tool calls proposed, denied, approved, executed, and reconciled;
- approval invalidations caused by resource changes;
- audit delivery lag, dropped events, and integrity checks;
- anomalous data volume, tenant switching, exports, and denied-resource enumeration;
- time to revoke a compromised principal across sessions, jobs, keys, and caches.

Avoid unbounded high-cardinality identity labels in metrics. Keep detailed identities in protected audit systems.

## Interview prompt

> Design identity, authorization, and tenant isolation for a construction operations assistant. Users can belong to several companies and projects; agents read documents and propose invoice approvals; workers and an ERP connector run asynchronously; permissions can be revoked; every consequential action must be auditable. Adapt the design for clinical records and emergency access.

Spend five minutes covering authentication, policy model, tenant resolution, enforcement layers, service identity, AI tools, approval, revocation, break glass, audit, and failure behavior.

## Worked answer

**One-line conclusion:** Treat identity as policy input, derive tenant scope from trusted state, and authorize every read and side effect outside the model with auditable least privilege.

“I would federate human authentication through OIDC and fully validate issuer, audience, signature, token type, expiry, state, nonce, and PKCE as applicable. External subjects map to internal principals and current tenant/project memberships. Authorization combines stable roles with resource relationships and contextual limits; it denies by default and returns a versioned internal decision. Tenant scope is resolved server-side and carried through typed context. The database uses tenant-scoped keys and optional row-level security, while caches, object URLs, search filters, queues, logs, and analytics enforce the same boundary. Each workload gets a short-lived service identity, and delegated calls preserve the originating subject to prevent confused-deputy behavior. The model has no ambient authority: typed tools re-check permission, state, resource version, approval, and idempotency before execution. Revocation updates policy versions and blocks future sensitive work. Break-glass access is strongly authenticated, time-limited, alerted, and reviewed. Audit records connect subject, tenant, policy, exact action plan, model/tool versions, and external outcome.”

## Practice lab

Design policy for these principals:

1. subcontractor uploading to one assigned project;
2. project manager reading RFIs and proposing an invoice approval;
3. controller approving within a financial limit;
4. auditor viewing finalized records;
5. support engineer investigating an incident;
6. AI worker executing an approved ERP operation.

For each, specify subject, tenant resolution, action, resource relationship, contextual condition, service identity, audit event, revocation behavior, and denial response.

Then trace this incident: a controller is removed from Project A after approving a plan but before the asynchronous worker executes it. Decide whether the job continues, which policy/version controls the decision, what the user sees, and what gets recorded.

## References

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [RFC 9700 — Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700)
- [NIST SP 800-207 — Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [PostgreSQL documentation — Row security policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [SMART App Launch — Scopes and launch context](https://hl7.org/fhir/smart-app-launch/)
