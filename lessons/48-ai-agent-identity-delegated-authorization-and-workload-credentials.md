# AI agent identity, delegated authorization, and workload credentials

An AI agent is not a user, and a model is not an identity provider.

When an agent retrieves data or calls a tool, the system must answer two different questions:

1. **Who is executing?** The agent service, workflow worker, connector, or tool gateway.
2. **On whose authority and for what purpose?** A user, service, approval, scheduled policy, or tenant-owned automation.

Collapsing those identities into one powerful API key creates a confused deputy: a low-privilege user can persuade a high-privilege agent to act beyond the user’s rights.

The safe design carries a verifiable, attenuated chain of authority from the initiating subject through every workload and tool boundary.

## Mental model

Represent an action as:

[
Decision = f(actor, subject, tenant, purpose, resource, action, constraints, evidence, time)
]

Where:

- **actor:** workload currently making the call;
- **subject:** user or service whose authority is being exercised;
- **tenant:** organization and isolation boundary;
- **purpose:** approved reason or workflow;
- **resource/action:** exact object and operation;
- **constraints:** amount, project, patient, region, tool, count, deadline, and risk;
- **evidence:** approval, policy, source versions, and delegation chain;
- **time:** issuance, expiry, revocation, and revalidation point.

A useful identity chain is:

user → application → agent orchestrator → workflow worker → tool gateway → external system.

Every hop authenticates the immediate caller and verifies the delegated subject/context. No hop trusts a tenant ID, role, approval, or identity produced by the model.

Authorization should **attenuate** downstream: each hop may keep or reduce authority, never silently expand it.

## When to use it

Use explicit delegated authorization when an AI system:

- acts on behalf of a signed-in user;
- runs asynchronous work after the user’s request ends;
- spawns sub-agents or workers;
- accesses multiple tools or external systems;
- serves multiple tenants;
- performs scheduled tenant-owned automation;
- requires human approval for consequential actions;
- uses regional or data-class restrictions;
- handles PHI, finance, contracts, security, or physical operations;
- must explain exactly who authorized an action.

A read-only prototype over public data may not need a full delegation service. It still needs a workload identity and must not store broad static credentials in prompts or code.

## Separate identity concepts

### Human identity

The user authenticates through an identity provider. Relevant claims may include:

- stable subject identifier;
- organization/tenant membership;
- authentication time and assurance;
- groups or roles;
- device/session context;
- consent or purpose where applicable.

Do not copy the entire identity token into every prompt or queue message. Derive the minimum trusted authorization context and use a downstream token intended for the specific service.

### Workload identity

Each running service, agent, worker, connector, and tool gateway needs its own cryptographic identity.

A workload identity answers “which deployed workload is calling?” It should be bound to:

- environment;
- service;
- deployment or trust domain;
- region;
- runtime/platform;
- optional workload measurement;
- certificate or key;
- lifecycle and revocation.

Use short-lived automatically rotated credentials instead of long-lived service-account secrets.

### Agent instance identity

A durable workflow should have a unique instance ID, but that ID is not sufficient authorization. Bind it to:

- tenant;
- initiating subject;
- workflow type and version;
- purpose;
- creation time;
- current state;
- delegation/approval references;
- policy revision;
- expiry;
- region;
- audit trace.

### Model identity

Record model/provider/release for evidence and policy, but do not grant authority to a model. The model cannot authenticate, hold a business role, or decide its own scopes.

### Tool identity

A tool service authenticates independently and verifies the caller, delegation, resource scope, and requested action. It must not assume “requests from the agent network are trusted.”

## Authentication, authorization, and delegation

- **Authentication:** proves an identity.
- **Authorization:** decides whether an identity may perform an action.
- **Delegation:** gives another actor limited authority to act for a subject.
- **Impersonation:** actor presents as the subject, often hiding the intermediary.
- **Approval:** a human or policy authorizes a specific proposed action.
- **Credential:** cryptographic material or token used to prove claims.
- **Capability:** authority to perform an operation, ideally narrow and transferable only under defined rules.

Prefer explicit delegation over impersonation. Audit should show both actor and subject.

A service credential proves the agent service exists; it does not prove the user authorized this invoice, patient record, or project action.

## Delegation token contents

A downstream token or authorization grant should include or reference:

- issuer;
- immediate actor;
- subject;
- tenant/organization;
- audience/resource server;
- permitted action/scopes;
- resource constraints;
- purpose/workflow;
- region/data-class constraints;
- delegation depth or parent reference;
- issued-at, not-before, and expiry;
- unique token/grant ID;
- session/workflow/approval ID;
- confirmation key for sender constraint where supported;
- policy version;
- revocation or introspection reference for high-risk use.

Keep tokens small. Complex resource lists can live in a server-side grant record identified by an opaque handle.

Never place raw source documents, model reasoning, or untrusted prompt text inside authorization tokens.

## Token exchange and audience

The application may exchange an incoming user token for a narrower token intended for the agent service. The agent may exchange again for a tool-specific token.

At each exchange:

1. authenticate the requesting workload;
2. validate the incoming token issuer, signature, time, audience, and subject;
3. load current tenant and policy state;
4. confirm the requested downstream audience;
5. intersect requested authority with caller and subject authority;
6. reduce scope, resource set, lifetime, region, and delegation depth;
7. record parent-child lineage;
8. issue a sender-constrained or otherwise protected token;
9. deny any attempted expansion.

A token minted for the retrieval service must not be accepted by the ERP tool. Validate audience at every resource server.

OAuth token exchange can express actor and subject relationships, but the exact security semantics are system-specific. Define them explicitly and test them.

## Sender-constrained credentials

A stolen bearer token can be replayed by anyone who possesses it.

Where feasible, bind tokens to a key held by the workload using:

- mutual TLS;
- proof-of-possession tokens;
- DPoP-style proof;
- platform/workload attestation;
- private key JWT or similar workload authentication.

The resource server verifies both token and key possession.

Sender constraint reduces replay but does not fix overbroad scopes, malicious authorized workloads, or leaked downstream data.

Protect keys in managed workload identity, HSM/KMS, or platform key stores. Do not place them in model context, environment dumps, source control, or queue payloads.

## Resource-level authorization

Scopes such as “read:documents” are often too broad. Add resource constraints:

- tenant and project;
- document IDs and versions;
- facility and room;
- patient and encounter;
- invoice/commitment/change-order IDs;
- maximum amount;
- approved recipient/domain;
- allowed tool;
- allowed operation;
- time window;
- region and data classification.

Resource servers enforce these constraints using trusted server-side IDs. The model may propose a resource name, but deterministic code resolves it within the authorized namespace.

Beware identifier collisions across tenants. A commitment ID of 123 is meaningless without tenant, source system, and project namespace.

## Purpose and context

Access may be valid only for a purpose:

- answer a project question;
- assemble a prior-authorization packet;
- reconcile a known ERP outcome;
- execute an approved change;
- run a scheduled compliance report.

Purpose should come from application/workflow policy, not free-form user or document text.

Context can include device, network, region, risk, authentication freshness, and workflow state. Treat these as authorization inputs with defined provenance.

For high-risk operations, require recent human authentication or step-up authentication.

## Tools as resource servers

Every tool endpoint should:

1. authenticate the immediate workload;
2. verify token issuer, audience, signature, time, and sender binding;
3. resolve actor, subject, tenant, and grant;
4. authorize exact resource and action;
5. validate typed arguments;
6. enforce rate, cost, and call limits;
7. recheck current policy and resource state;
8. require approval evidence when applicable;
9. generate or accept an idempotency key;
10. record decision and external outcome;
11. return structured status without unnecessary sensitive data.

The agent cannot grant itself tools by adding names to a prompt. Tool availability comes from trusted policy.

A model’s tool call is untrusted input to the tool gateway.

## Confused-deputy prevention

A confused deputy has powerful credentials and is tricked into using them for an unauthorized requester.

Defenses:

- carry the initiating subject and actor separately;
- derive tenant from authenticated context;
- issue audience-specific tokens;
- constrain resources and actions;
- use narrow workload identities;
- have the resource server enforce authorization;
- prohibit arbitrary URLs and resource IDs;
- bind approvals to exact payloads;
- log the full delegation chain;
- test overlapping identifiers across tenants.

Do not solve this by asking the model “is this user authorized?” The model lacks authoritative identity and policy state.

## Approval-bound authority

A human approval should create a narrow authorization grant, not a boolean flag.

Bind the approval to:

- actor/subject and tenant;
- workflow and proposal ID;
- exact tool;
- canonical action payload digest;
- target resource and source system;
- amount/limits;
- evidence/source versions;
- policy version;
- approver identity and authority;
- approval time and expiry;
- idempotency key;
- required revalidation conditions.

Before execution:

1. authenticate worker;
2. verify grant and sender binding;
3. confirm payload digest;
4. check approval is unused/valid;
5. recheck approver and subject authority where policy requires;
6. re-read authoritative target state;
7. detect material changes;
8. execute idempotently;
9. reconcile uncertain external outcome;
10. consume or close the grant.

An approval does not allow the model to edit the payload afterward.

## Long-running and asynchronous work

A user’s browser session may end while a workflow continues.

Choose deliberately among:

- **subject-bound continuation:** work stops when subject access is revoked;
- **snapshot grant:** approved job keeps narrow authority until expiry;
- **tenant-owned automation:** policy, not an individual session, owns the action;
- **service operation:** platform maintenance runs under a platform identity.

Record which model applies.

Queue messages should carry an opaque workflow/grant reference, not a reusable broad bearer token. At execution, the worker obtains a fresh tool token after current checks.

Long-running workflows need:

- grant expiry and renewal;
- cancellation;
- revocation;
- policy/version recheck;
- source freshness;
- subject/approver status rules;
- regional continuity;
- retry and outcome reconciliation.

If authorization infrastructure is unavailable, read-only cached decisions may be acceptable under bounded policy; consequential writes should normally queue or fail closed.

## Sub-agents and delegation depth

Multi-agent systems add hops.

A parent agent may delegate only a subset of:

- tools;
- resources;
- actions;
- time;
- token/cost budget;
- network destinations;
- data classifications;
- regions;
- call depth.

The child receives a new identity/grant showing the parent actor and original subject. It cannot copy the parent token and cannot mint siblings.

Set maximum delegation depth and prevent cycles. Record a delegation DAG:

- parent grant;
- child workload;
- requested and issued authority;
- reason/task;
- expiry;
- result;
- revoked/closed state.

A child’s output is untrusted data when it returns to the parent.

## Scheduled agents

A scheduled job may have no active human.

Define a tenant-owned automation principal with:

- named owner;
- approved workflow;
- allowed schedule;
- resource scope;
- tool set;
- budget;
- data class/region;
- maximum action risk;
- review requirements;
- expiry/renewal;
- emergency disable.

Do not use the creator’s permanent user token. If the owner leaves, an explicit ownership-transfer or disable process should occur.

Separate “may draft” from “may execute.” Many scheduled agents should only prepare a queue for human review.

## External systems and legacy APIs

Some ERP, project, or clinical systems support only one integration account.

Wrap them with a tool gateway that:

- authenticates each internal caller;
- enforces subject/tenant/resource policy;
- maps to the shared integration credential;
- limits operations;
- applies idempotency;
- records actor and subject;
- reconciles outcomes;
- prevents direct network access from agents.

The external audit may show only the integration account, so preserve high-quality internal evidence and include supported correlation IDs or comments.

Where the external system supports delegated OAuth, prefer user or workload-specific delegated access with narrow scopes.

## Credential lifecycle

### Issuance

Issue after verifying identity, policy, audience, and resource scope. Prefer short lifetimes.

### Storage

Use in-memory or platform credential stores. Never store tokens in prompts, conversation memory, vector indexes, traces, or model-visible scratchpads.

### Rotation

Automate workload certificate/key rotation. Overlap safely, test consumers, and revoke old material.

### Revocation

Support:

- user/tenant membership change;
- workload compromise;
- approval cancellation;
- tool disable;
- incident response;
- model/workflow release withdrawal;
- owner departure.

Short-lived tokens reduce revocation delay. High-risk grants may need introspection or server-side state.

### Destruction

Clear credentials from memory and temporary files. Ensure crash dumps and support bundles exclude them.

## Policy decision and enforcement

Separate:

- **policy information point:** identity, groups, resource state, purpose, risk, consent;
- **policy decision point:** evaluates policy;
- **policy enforcement point:** gateway, retrieval service, tool, database, or external adapter;
- **policy administration point:** versioned policy authoring and release.

Enforce near the resource. A central decision service helps consistency, but each resource server must fail safely if decision context is absent.

Cache decisions only with:

- exact actor/subject;
- tenant;
- resource/action;
- purpose;
- policy version;
- time and revocation assumptions;
- bounded TTL.

Never cache a broad “user is admin” result and reuse it across projects.

## Data-plane architecture

A practical request flow:

1. user authenticates to the application;
2. application derives tenant and session context from trusted identity;
3. authorization service issues a narrow agent grant;
4. orchestrator starts a workflow under its workload identity;
5. retrieval service receives an audience-specific read token;
6. authorized evidence is assembled and labeled as untrusted content;
7. model proposes a typed tool action;
8. deterministic policy resolves resource IDs and checks proposal eligibility;
9. high-impact proposal enters human review;
10. approval service creates a payload-bound grant;
11. worker exchanges it for a short-lived tool token;
12. tool gateway revalidates and calls the external system;
13. outcome is recorded and reconciled;
14. grants close or expire;
15. audit links user, actors, model, evidence, approval, tool, and outcome.

The model never sees credentials and never chooses the authenticated tenant.

## Multi-region and provider boundaries

Identity and authorization metadata can itself be sensitive.

Define:

- home region for tenant/grants;
- approved token issuers and signing keys by region;
- cross-region trust;
- replication of revocation state;
- failover semantics;
- clock-skew tolerance;
- regional audit;
- provider account eligibility.

A token issued in one region should not automatically work everywhere. Resource servers validate region and audience.

During partition, use short-lived regional grants and conservative behavior. Do not allow a failover region to broaden tool or data access because the primary policy service is unavailable.

External model providers should not receive tool credentials or reusable delegated tokens. They receive only the minimum prompt/context through the AI gateway.

## Audit and non-repudiation

For every consequential action record:

- initiating subject;
- immediate actor at each hop;
- tenant;
- authentication context;
- delegation/grant IDs and lineage;
- policy decision and version;
- resource/action;
- canonical payload digest;
- evidence and source versions;
- model/provider/release;
- approval identity/time;
- token audience and key-binding identifier;
- tool attempt and idempotency key;
- external result or outcome-unknown state;
- reconciliation;
- revocation/cancellation events.

Avoid logging raw tokens or secrets. Hashing a token is often unnecessary; log stable grant IDs and key thumbprints designed for audit.

Audit evidence must be access-controlled and retained by purpose.

## Observability and detection

Monitor:

- token exchanges and denials;
- attempted scope/resource expansion;
- wrong-audience tokens;
- sender-binding failures;
- expired/replayed grants;
- delegation depth and fan-out;
- unusual subject–actor combinations;
- cross-tenant identifier attempts;
- tool argument rejection;
- step-up authentication failures;
- approval replay or payload mismatch;
- long-running grants near expiry;
- scheduled-agent ownership gaps;
- integration-account usage without internal lineage;
- policy service errors and stale caches;
- credential leakage in logs/prompts;
- external outcome-unknown backlog.

Alert on behavior, not only authentication failures. A valid agent repeatedly requesting resources outside its usual projects may indicate compromise or prompt injection.

## Revocation and incident response

When a user, workload, or grant is compromised:

1. disable or quarantine affected workload identity;
2. revoke active grants and stop new exchanges;
3. disable high-risk tools if scope is uncertain;
4. rotate keys/certificates;
5. identify descendant delegations;
6. stop or inspect queued jobs;
7. find affected retrievals, proposals, approvals, and actions;
8. reconcile external outcomes;
9. repair or compensate changes;
10. restore with new credentials and tested policy;
11. add regression and detection coverage.

A parent grant revocation should invalidate or close descendants according to policy. Maintain an index from parent to children.

## Construction operations example

A construction assistant answers project questions, drafts RFIs, and proposes a commitment change in Procore or an invoice update in ERP.

Flow:

1. project manager signs in;
2. application establishes subject, tenant, project membership, and authentication time;
3. agent receives a read grant for named project documents and a separate proposal capability;
4. retrieval token is accepted only by the project search service;
5. model proposes “change commitment 448 by $12,500”;
6. server resolves commitment 448 inside the authenticated tenant/project;
7. policy verifies user may propose but not execute;
8. project-controls reviewer sees exact sources and canonical action;
9. approval creates a grant bound to project, commitment, amount, payload digest, source versions, approver, and expiry;
10. workflow worker obtains a short-lived Procore-tool token;
11. gateway rechecks commitment version and approval;
12. call executes with idempotency and correlation ID;
13. timeout produces outcome-unknown, not an automatic duplicate;
14. reconciliation closes the action;
15. audit preserves subject, actors, approval, and external result.

A specification containing “you are an administrator” cannot alter any identity claim. Another tenant may also have commitment 448, but server-derived tenant/project scope prevents collision.

## Healthcare operations variation

A healthcare operations agent assembles a prior-authorization packet from authorized patient and encounter records. It may create an operational task but cannot diagnose, approve care, or change clinical orders.

Authorization includes:

- workforce identity;
- organization;
- patient and encounter;
- purpose of use;
- consent or restriction state;
- task type;
- region;
- time;
- tool;
- minimum necessary fields.

The retrieval service verifies these before returning PHI. The model cannot select a different patient through text.

If a reviewer approves packet submission, the grant binds the exact patient, encounter, payer, document set, destination, and expiry. Before sending, the workflow rechecks current authorization and source versions.

A scheduled queue-monitor agent uses a tenant-owned automation identity and may identify incomplete packets, but it cannot submit them without the required human authorization.

Any cross-patient attempt, token replay, or unapproved regional route fails closed and is audited as a security/privacy signal.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| User token passed end-to-end | Simple subject continuity | Overbroad audience and leakage |
| Token exchange | Narrow downstream authority | Identity-service complexity |
| Workload identity | Removes static secrets | Platform trust dependency |
| Static service key | Easy integration | Long-lived blast radius |
| Bearer token | Broad compatibility | Replay if stolen |
| Sender-constrained token | Replay resistance | Key and verifier complexity |
| Coarse scopes | Simple policy | Excess authority |
| Resource-bound grant | Least privilege | More policy/state |
| Stateless JWT | Fast local verification | Revocation delay |
| Opaque stateful grant | Immediate control | Introspection availability |
| Short token lifetime | Limits compromise | Refresh/exchange load |
| Long-running snapshot grant | Workflow continuity | Authority can outlive user state |
| Revalidate every step | Fresh authorization | Latency/dependency |
| Payload-bound approval | Prevents mutation | Reapproval after change |
| Broad integration account | Works with legacy systems | Confused-deputy risk |
| Tool gateway | Central enforcement/audit | Added component and latency |
| Tenant-owned automation | Survives user departure | Governance burden |
| Per-user delegation | Clear accountability | Breaks when user unavailable |
| Central policy decision | Consistency | Shared dependency |
| Resource-local enforcement | Strong final boundary | Distributed implementation |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Model treated as identity | Prompt changes authority | Trusted identity plane |
| Agent service identity used as user authority | Confused deputy | Separate actor and subject |
| Tenant ID accepted from model | Cross-tenant access | Server-derived scope |
| User token sent to every service | Token theft and excess audience | Token exchange |
| Audience not validated | Token reused at wrong tool | Resource indicators/check |
| Broad “admin” scope | Excess resource access | Resource-bound grants |
| Child gets parent token | Delegation expansion/replay | New attenuated child grant |
| No delegation depth | Agent spawning explosion | Depth/fan-out limits |
| Bearer token in queue | Replay after theft | Opaque grant reference/fresh exchange |
| Token in prompt or memory | Model may reveal it | Credential isolation |
| Static credential in connector | Large persistent blast radius | Workload identity/rotation |
| Shared integration account trusted directly | User bypasses policy | Tool gateway |
| Approval stored as boolean | Payload changes after approval | Digest-bound grant |
| Approval never expires | Stale action executes | TTL and state recheck |
| Approver loses access | Invalid authority persists | Defined revalidation rule |
| Subject access revoked mid-job | Unauthorized continuation | Grant semantics/revocation |
| Scheduled job uses creator token | Breaks or overreaches | Automation principal |
| Tool trusts internal network | Compromised service acts freely | Authenticate every hop |
| Policy check only at orchestrator | Tool bypass possible | Resource-server enforcement |
| Token cache omits resource/policy | Decision reused incorrectly | Exact cache key and TTL |
| Region omitted from grant | Cross-region policy breach | Regional claim/enforcement |
| Clock skew too permissive | Expired token accepted | Bounded skew and secure time |
| Parent revoked, child remains | Descendant authority persists | Delegation lineage revocation |
| Audit logs raw credentials | Secondary compromise | Grant IDs, no tokens |
| Outcome-unknown retried blindly | Duplicate side effect | Idempotency/reconciliation |

## What to measure

Track by tenant, actor, subject, workflow, tool, and region:

- workload-identity issuance and rotation health;
- static credential inventory and age;
- token exchange latency, failure, and denied expansion;
- wrong-audience and sender-binding failures;
- grant scope/resource size and lifetime;
- delegation depth, fan-out, and descendant count;
- cross-tenant resource attempts;
- authorization decision latency and cache age;
- expired, replayed, revoked, and payload-mismatched grants;
- approval-to-execution time and revalidation failures;
- long-running jobs canceled after revocation;
- scheduled agents without active owner/review;
- tool calls with complete subject/actor lineage;
- legacy integration calls lacking correlation;
- credential findings in prompts, memory, queues, logs, and dumps;
- outcome-unknown actions and reconciliation age;
- incident descendant-grant discovery time;
- percentage of tools enforcing audience, resource, and action locally.

A good metric is not “all requests are authenticated.” It is “every resource operation can prove the immediate actor, original authority, exact allowed action, and current decision.”

## Interview prompt

> Design identity and delegated authorization for a multi-tenant AI agent that reads project or patient records, spawns sub-agents, runs asynchronous workflows, calls legacy and OAuth-enabled tools, supports scheduled automations, and performs approval-bound writes. Prompts and retrieved documents are untrusted.

Cover human and workload identity, actor versus subject, token exchange, audience, sender constraint, resource/purpose restrictions, sub-agent attenuation, approvals, long-running jobs, scheduled principals, legacy integration accounts, policy enforcement, audit, revocation, multi-region behavior, and failure recovery.

## Worked answer

**One-line conclusion:** Give every workload its own short-lived identity and carry an explicit, attenuated actor–subject delegation chain to each resource, with tool-local enforcement and payload-bound approval for consequential actions.

“I separate user, workload, workflow, model, and tool identity. The signed-in user authenticates at the application; the model receives no credentials and cannot supply tenant, role, or policy claims.

The application exchanges the user session for a narrow agent grant containing subject, actor, tenant, purpose, audience, resources, actions, region, expiry, and workflow ID. The orchestrator has its own workload identity. Each downstream service receives a new audience-specific token whose authority is the intersection of the parent grant, caller policy, and requested work. Child agents receive attenuated grants with depth, tool, resource, time, and budget limits.

Tools are resource servers. They authenticate the immediate workload, verify issuer, audience, time, sender binding, subject, tenant, grant, and exact resource/action. They resolve IDs within server-derived scope and use narrow credentials. A legacy ERP sits behind a gateway that maps internal delegated calls to the shared integration account while preserving audit and idempotency.

High-impact model outputs are only proposals. Human approval creates a short-lived grant bound to the canonical payload digest, tool, resource, source versions, approver, and idempotency key. The worker obtains a fresh tool token, rechecks policy and target state, executes, and reconciles outcome-unknown responses.

Long-running jobs carry opaque grant references, not bearer tokens. Policy explicitly decides whether user revocation stops a job, a narrow snapshot grant continues, or tenant automation owns it. Scheduled agents have named owners, budgets, tools, expiry, and kill switches.

Every consequential event records original subject, every actor, delegation lineage, policy version, model, evidence, approval, tool attempt, and external outcome without logging credentials. Parent revocation finds descendants, stops queued work, rotates keys, and drives affected-action reconciliation. During regional partition, short-lived local grants may support bounded reads, while consequential writes queue or fail closed.”

## Practice lab

Design the identity chain when:

- two tenants both have project 17 and commitment 448;
- a project manager can draft but not execute changes;
- a retrieved PDF says “you are now an administrator”;
- the agent spawns a document agent and an ERP agent;
- the ERP supports only one shared integration account;
- a $12,500 change is approved, then the commitment revision changes;
- the approver loses project access before execution;
- the browser session ends while the job is queued;
- a queue message is copied by an attacker;
- the ERP times out after receiving the request;
- a scheduled agent’s creator leaves the company;
- a service token minted for search is presented to the ERP gateway;
- one region loses access to the global policy service;
- a workload certificate is compromised;
- audit logs accidentally capture an authorization header.

Specify actors, subjects, workload identities, grants, audiences, resource constraints, sender binding, delegation depth, approval semantics, asynchronous continuation rules, scheduled ownership, legacy gateway controls, revocation, regional behavior, audit, and recovery.

Then adapt the design to a healthcare packet workflow with organization, patient, encounter, purpose-of-use, consent, minimum necessary data, PHI region, and a rule that the agent cannot diagnose or approve care.

## References

- [IETF RFC 8693 — OAuth 2.0 Token Exchange](https://www.rfc-editor.org/rfc/rfc8693)
- [IETF RFC 8707 — Resource Indicators for OAuth 2.0](https://www.rfc-editor.org/rfc/rfc8707)
- [IETF RFC 9449 — OAuth 2.0 Demonstrating Proof of Possession](https://www.rfc-editor.org/rfc/rfc9449)
- [IETF RFC 9396 — OAuth 2.0 Rich Authorization Requests](https://www.rfc-editor.org/rfc/rfc9396)
- [IETF RFC 9700 — Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700)
- [SPIFFE specifications](https://spiffe.io/docs/latest/spiffe-about/overview/)
- [NIST SP 800-207 — Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [NIST SP 800-63 — Digital Identity Guidelines](https://pages.nist.gov/800-63-4/)
