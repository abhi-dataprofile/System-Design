# 03 — Relational data modeling and indexes

Previous: [API boundaries and request lifecycle](02-api-boundaries-and-request-lifecycle.md). Study time: 35–45 minutes including the exercise.

## Mental model

A relational model turns business rules into data structures the database can enforce. Tables describe entities and events; keys establish identity; constraints reject impossible states; transactions make related changes atomic; indexes provide efficient access paths.

Start with invariants, not columns:

- Every project belongs to exactly one tenant.
- An invoice belongs to a project in the same tenant.
- A source invoice version can receive at most one effective approval.
- An approval operation records who requested it, what version they reviewed, and its current outcome.
- Tenant A must never reference or retrieve Tenant B's records.

Application checks produce helpful errors. Database constraints remain the final protection against concurrent requests, retries, and programming mistakes.

## When a relational database fits

Use a relational database when the workflow has structured records, relationships, transactions, uniqueness rules, and several query paths. Invoice approvals, clinical work queues, access grants, and audit metadata usually have these properties.

A document store may fit aggregates with flexible shapes and few cross-record rules. Object storage fits large PDFs and images. Search engines fit ranked text retrieval. These systems can complement the relational source of operational truth; copying data into them creates freshness and reconciliation work.

## Model the construction workflow

The simplified PostgreSQL example uses tenant-scoped composite keys. UUIDs may be globally unique, but including `tenant_id` in keys and foreign keys lets the database enforce same-tenant relationships.

```sql
CREATE TABLE tenants (
    tenant_id uuid PRIMARY KEY,
    name text NOT NULL
);

CREATE TABLE projects (
    tenant_id uuid NOT NULL REFERENCES tenants(tenant_id),
    project_id uuid NOT NULL,
    name text NOT NULL,
    PRIMARY KEY (tenant_id, project_id)
);

CREATE TABLE invoices (
    tenant_id uuid NOT NULL,
    invoice_id uuid NOT NULL,
    project_id uuid NOT NULL,
    vendor_id uuid NOT NULL,
    source_version text NOT NULL,
    status text NOT NULL CHECK (status IN ('open', 'approved', 'rejected', 'void')),
    amount_cents bigint NOT NULL CHECK (amount_cents >= 0),
    currency char(3) NOT NULL,
    due_at timestamptz,
    updated_at timestamptz NOT NULL,
    PRIMARY KEY (tenant_id, invoice_id),
    FOREIGN KEY (tenant_id, project_id)
        REFERENCES projects (tenant_id, project_id),
    UNIQUE (tenant_id, invoice_id, source_version)
);

CREATE TABLE approval_operations (
    tenant_id uuid NOT NULL,
    operation_id uuid NOT NULL,
    invoice_id uuid NOT NULL,
    reviewed_source_version text NOT NULL,
    requested_by uuid NOT NULL,
    idempotency_key text NOT NULL,
    request_fingerprint text NOT NULL,
    state text NOT NULL CHECK (
        state IN ('pending', 'running', 'reconciling', 'succeeded', 'failed')
    ),
    created_at timestamptz NOT NULL,
    completed_at timestamptz,
    PRIMARY KEY (tenant_id, operation_id),
    FOREIGN KEY (tenant_id, invoice_id, reviewed_source_version)
        REFERENCES invoices (tenant_id, invoice_id, source_version),
    UNIQUE (tenant_id, requested_by, idempotency_key)
);
```

The three-column invoice uniqueness constraint is required because the operation's composite foreign key references those three columns. The invoice primary key still prevents duplicate invoice IDs within a tenant.

This schema records the version the manager reviewed. It does not by itself guarantee that the invoice is still on that version when the ERP mutation occurs. The execution path must perform a conditional update against the current source version, as discussed in Lesson 02.

### Money, time, and state

- Store money as an exact integer in the smallest currency unit or as a deliberately bounded decimal. Floating point is unsuitable for financial equality.
- Store a currency code with the amount. Never add amounts across currencies without a defined conversion policy.
- Store instants as timezone-aware timestamps and preserve the business timezone separately when rules depend on local dates.
- A `CHECK` constraint is useful for a small stable state set. A transition such as `succeeded → running` requires transaction logic, a database function, or a guarded update; membership in the set alone does not validate transitions.
- Keep large invoice PDFs in object storage and store an object identifier, checksum, media type, and authorization metadata in the database. Large binary payloads make common rows and backups heavier.

## Design indexes from queries

An index has value when it supports an important query or constraint. Each index consumes storage and adds work to writes, vacuuming, and cache use.

Suppose the main work-queue query is:

```sql
SELECT invoice_id, vendor_id, amount_cents, due_at
FROM invoices
WHERE tenant_id = $1
  AND project_id = $2
  AND status = 'open'
ORDER BY due_at, invoice_id
LIMIT 50;
```

A matching B-tree index is:

```sql
CREATE INDEX invoices_open_queue_idx
ON invoices (tenant_id, project_id, status, due_at, invoice_id);
```

Equality-filtered columns lead, followed by the range or ordering columns. The final `invoice_id` makes pagination order deterministic. This index is effective for the full query and useful for some prefixes; it is not a universal index for queries that omit the leading tenant/project fields.

For a large table where most invoices are closed, a partial index can reduce size:

```sql
CREATE INDEX invoices_open_due_idx
ON invoices (tenant_id, project_id, due_at, invoice_id)
WHERE status = 'open';
```

The query predicate must allow the planner to establish that the partial-index predicate applies. Parameterized or differently expressed predicates can affect that proof, so verify with the actual query plan.

For operation monitoring:

```sql
CREATE INDEX approval_pending_age_idx
ON approval_operations (tenant_id, state, created_at, operation_id);
```

This supports tenant-scoped pending-operation scans ordered by age. If workers scan across every tenant, design a separate global queue access pattern and ensure worker authorization is explicit.

### Do not guess: inspect the plan

Use `EXPLAIN (ANALYZE, BUFFERS)` on representative, non-sensitive test data or in a controlled production diagnostic process. Check:

- estimated rows versus actual rows;
- sequential scan versus index scan;
- rows removed by filters;
- sort operations and memory/disk use;
- buffer hits and reads;
- end-to-end query latency under realistic concurrency.

`EXPLAIN ANALYZE` executes the statement. Wrap write experiments in a rollback-safe transaction only when all external effects and database behaviors are understood; use read-only copies for risky investigation.

## Transactions and concurrent updates

At PostgreSQL's default Read Committed isolation, each statement sees a snapshot at statement start. A read followed by a later write is not automatically one atomic decision. Two approvers can both read `open` before either commits.

Use a guarded update:

```sql
UPDATE invoices
SET status = 'approved',
    updated_at = now()
WHERE tenant_id = $1
  AND invoice_id = $2
  AND status = 'open'
  AND source_version = $3
RETURNING invoice_id, status;
```

Exactly one successful transition returns a row. Zero rows means the record was missing, unauthorized under the scoped query, already changed, or on another version; the service resolves that distinction without leaking data.

For rules spanning several rows, choose among:

- a transaction plus row locks in a consistent order;
- a uniqueness or exclusion constraint;
- a higher isolation level with retry handling;
- redesigning the invariant so one guarded statement or constraint can enforce it.

Serializable isolation can reject a transaction to prevent an outcome inconsistent with serial execution. The application must recognize serialization failures and retry the entire transaction with a bounded policy. Raising isolation does not fix external API side effects inside a database transaction.

## Tenant isolation

Every tenant-owned access path should carry verified tenant context. Composite foreign keys stop cross-tenant references, and tenant-leading indexes make scoped queries efficient. They do not automatically add `tenant_id` to application queries.

Defense layers can include:

1. deriving tenant memberships from authenticated identity;
2. repository/service methods that require tenant context;
3. composite constraints;
4. PostgreSQL row-level security for suitable deployments;
5. tests that attempt cross-tenant reads and writes;
6. audit events for sensitive access.

Row-level security is powerful but policy, ownership, connection-role, and session-context mistakes can create gaps. Treat it as a designed security boundary with dedicated tests, not a checkbox. A fuller security lesson will cover it.

## Healthcare variation

For a clinical work queue, replace projects and invoices with facilities, patients/encounters, and review tasks. Keep PHI out of indexes and audit payloads unless the workflow truly requires it. A task may reference a patient record, but authorization depends on purpose, role, facility, and current relationship; a foreign key proves data integrity, not permission.

For frequently accessed FHIR resources, store the source resource ID and version identifier. Before a consequential update, compare the version and apply a conditional source write when the FHIR server supports it. The local database cannot independently guarantee that an external clinical record remained unchanged.

## Tradeoffs and failure modes

- **Normalize versus duplicate:** Normalization reduces inconsistent copies. Carefully chosen denormalized fields can make reads simpler, but every copy needs an owner and update strategy.
- **Natural versus surrogate keys:** Natural keys expose business meaning but can change. Surrogate IDs are stable, while separate unique constraints preserve business uniqueness.
- **Too few indexes:** Important queries scan and sort excessive data. **Too many indexes:** writes slow, storage grows, and maintenance consumes resources.
- **Soft deletion:** A `deleted_at` flag preserves history but every active-record query and uniqueness rule must account for it. Partial unique indexes can help when the exact lifecycle is defined.
- **Unbounded tables:** Approval operations and audit events grow continuously. Define retention, archival, partitioning criteria, and deletion rules before volume becomes an incident.
- **Read replicas:** They scale suitable reads but can lag. Never send a read-after-write status check to a replica unless the product accepts stale results or uses a consistency mechanism.
- **External drift:** Database success and ERP success are separate facts. Persist external references and reconciliation state.

## Interview prompt

> Design the relational data model and indexes for a multi-tenant construction platform where managers review invoices by project, approve a specific invoice version, and monitor pending approval operations. Prevent cross-tenant references and handle two managers approving concurrently.

Spend five minutes describing invariants, schema, query patterns, indexes, and the concurrency strategy.

## Worked answer

**One-line conclusion:** Encode tenant and workflow invariants as constraints, then build the smallest indexes that match measured query patterns.

“I would model tenants, projects, invoices, and approval operations. Tenant-owned tables use composite keys beginning with tenant ID, and foreign keys include tenant ID so an invoice cannot reference another tenant's project. The operation records the reviewed source version, actor, request fingerprint, and state, with a tenant-and-actor-scoped idempotency uniqueness constraint. For the open-invoice queue, I would index tenant, project, status, due date, and invoice ID in query order, or use a measured partial index for open rows. Approval is a guarded update on tenant, invoice, open status, and source version; only one concurrent request can change the row. I would inspect actual plans and contention before adding indexes, test cross-tenant references and concurrent approvals, and keep external ERP reconciliation separate from the local transaction.”

## Practice lab

For each query below, propose an index or explain why the primary key is enough:

1. Fetch one invoice by tenant and invoice ID.
2. List 50 open invoices for one project ordered by due date.
3. Find pending operations for one tenant older than ten minutes.
4. Look up an operation by tenant and operation ID.
5. Produce a monthly cross-tenant finance report.

Then explain which query must use the primary database after an approval and why.

Expected direction: primary keys serve 1 and 4; a queue index serves 2; a state/age index serves 3; query 5 needs an explicit analytics path rather than weakening tenant-scoped operational access. The immediate post-approval read needs an authoritative or consistency-aware path because a replica may lag.

## References

- [PostgreSQL: Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- [PostgreSQL: Multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
- [PostgreSQL: Partial indexes](https://www.postgresql.org/docs/current/indexes-partial.html)
- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL: Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL: Row security policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
