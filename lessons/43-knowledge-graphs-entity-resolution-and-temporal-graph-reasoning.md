# Knowledge graphs, entity resolution, and temporal graph reasoning

Operational questions often cross many systems:

- Which invoices depend on this change order?
- Which drawing revision supports this RFI response?
- Which subcontractor, vendor, and ERP supplier records refer to the same organization?
- What was believed to be true when an approval was granted?

A knowledge graph can connect these records, but only if identity, provenance, authorization, and time are modeled explicitly.

## Mental model

Treat a knowledge graph as a **versioned network of evidence-backed claims**, not a magical source of truth.

A graph contains:

- **entities** — project, organization, contract, invoice, change order, RFI, drawing, person, equipment, patient, encounter;
- **relationships** — belongs to, references, supersedes, approves, supplies, affects, derived from;
- **properties** — typed attributes;
- **provenance** — source, record, extractor, reviewer, and confidence;
- **valid time** — when the claim applies in the business world;
- **system time** — when the platform learned or stored it;
- **authorization scope** — who may see the entity or relationship.

Keep source records and resolved entities separate.

```text
source record ──asserts identity/relationship──> canonical entity
      |                                      |
 provenance + time                     stable graph ID
```

An ERP vendor row, a contract company name, and an email domain may all be evidence about one organization. Do not destroy those source identities by merging rows in place.

The central invariant is:

> Every graph fact and entity match can be traced to source evidence, confidence, decision policy, validity interval, and reversible resolution history.

## When to use it

Use a graph when questions depend on multi-hop relationships, evolving identity, or heterogeneous sources:

- impact analysis across documents and workflows;
- entity resolution across ERP, project-management, email, and document systems;
- dependency and lineage queries;
- temporal “as of” reasoning;
- fraud or conflict networks;
- recommendations based on relationships;
- graph-assisted retrieval;
- equipment, facility, patient, or provider networks;
- root-cause and affected-output analysis.

Use relational tables when the schema and joins are stable, transactions and constraints dominate, and graph traversals add little value.

Use search when the primary need is relevance over text. Use a graph to constrain, connect, and explain retrieval—not to replace full-text or vector indexes.

Do not add a graph merely because an architecture diagram looks sophisticated. A graph creates ontology, identity, ingestion, query, and governance work.

## Source of truth and graph projection

Classify data:

### Authoritative records

Examples: approved contract, ERP invoice, FHIR patient identifier, signed change order.

The graph points to these records and their versions. It does not silently override them.

### Derived facts

Examples: invoice references change order, drawing affects room, vendor aliases resolve to organization.

Derived facts require provenance, version, confidence, and possibly review.

### Hypotheses

Examples: two companies may be the same, an RFI may explain a cost variance.

Store as candidate relationships with explicit state. Do not present them as fact.

### Policy or ontology

Definitions of entity types, relationship semantics, cardinality, and constraints.

Version these like code.

## Ontology and schema

Start from questions and invariants, not from every noun in the business.

For each node type define:

- stable identifier;
- tenant/organization scope;
- authoritative source;
- required properties;
- sensitivity;
- lifecycle;
- allowed relationships;
- temporal behavior.

For each relationship define:

- direction and meaning;
- source and target types;
- cardinality;
- whether it is asserted, inferred, or reviewed;
- valid/system time;
- confidence;
- evidence requirements;
- authorization propagation;
- supersession and deletion.

Examples:

```text
(Project)-[:HAS_CONTRACT]->(Contract)
(InvoiceLine)-[:CLAIMS_AGAINST]->(Commitment)
(ChangeOrder)-[:MODIFIES]->(Contract)
(RFI)-[:REFERENCES]->(DrawingRevision)
(DrawingRevision)-[:SUPERSEDES]->(DrawingRevision)
(OrganizationRecord)-[:RESOLVES_TO]->(Organization)
```

Avoid vague edges such as `RELATED_TO`. They are easy to create and hard to interpret.

A schema constraint can reject impossible relationships, but semantic validity still depends on source evidence and time.

## Identity model

Use three identity layers:

1. **source record ID** — immutable within the source system;
2. **canonical entity ID** — stable graph identity;
3. **business identifiers** — tax ID, vendor code, contract number, NPI, MRN, email, phone.

Business identifiers may change, collide, or be reused. Store namespace, issuer, validity, and confidence.

For multi-tenant data, source keys include tenant and source:

```text
(tenant_id, source_system, source_record_id)
```

Never assume vendor `123` in two tenants is the same entity.

A canonical ID should survive alias and property changes. Merges and splits create resolution events rather than rewriting history.

## Entity resolution pipeline

Entity resolution determines which records refer to the same real-world entity.

### 1. Normalize

- case and Unicode;
- punctuation and legal suffixes;
- addresses and postal standards;
- phone and email;
- dates;
- names and abbreviations;
- identifiers with issuer/namespace.

Preserve raw values. Normalization is a candidate-generation aid, not proof.

### 2. Block candidates

Avoid all-pairs comparison. Generate plausible candidates using:

- exact identifier;
- normalized name and postal code;
- phone/email/domain;
- address fragments;
- phonetic or n-gram keys;
- embedding similarity;
- graph neighborhood;
- temporal/geographic constraints.

Use multiple blocking rules to preserve recall. Monitor candidates missed by blocking.

### 3. Score pairs or clusters

Signals:

- exact authoritative identifier;
- name/address similarity;
- shared contacts;
- temporal overlap;
- source reliability;
- contradictory identifiers;
- graph consistency;
- known aliases;
- negative evidence.

A score is not identity. Calibrate it on representative adjudicated pairs.

### 4. Decide

Use thresholds:

- auto-match above a high threshold when policy permits;
- human review in the ambiguous band;
- no-match below a low threshold;
- mandatory review for high-risk entities.

Require stronger evidence for patient or payment identity than for search suggestions.

### 5. Cluster

Pairwise matches may create inconsistent clusters:

- A matches B;
- B matches C;
- A contradicts C.

Do not assume transitivity blindly. Enforce cluster constraints and review conflicts.

### 6. Persist resolution history

Record model/rules, features, evidence, confidence, reviewer, timestamp, and prior cluster. Support merge, unmerge, and split.

## Deterministic and probabilistic matching

Deterministic rules:

- exact tax ID with valid namespace;
- exact source cross-reference;
- contract-defined vendor mapping.

Benefits: explainable and precise.  
Risks: identifiers can be missing, mistyped, reused, or fraudulent.

Probabilistic methods estimate match likelihood from multiple fields. The classic Fellegi–Sunter view compares how likely field agreements are under match versus non-match.

Benefits: handles noisy data.  
Risks: calibration, correlated fields, population differences, and threshold governance.

Combine them: deterministic high-trust links, probabilistic candidates, negative constraints, and human review.

Never let a language model create canonical identity solely from conversational similarity.

## Merge, unmerge, and survivorship

Do not physically collapse source records.

A canonical entity maintains links to source records and a **survivorship policy** for display:

- authoritative source priority;
- most recent valid value;
- reviewer-confirmed value;
- source-specific value by context.

For example, legal name may come from a contract/master record while contact phone comes from the latest verified directory.

A merge operation should be reversible. Store:

- prior canonical IDs;
- affected records and relationships;
- reason/evidence;
- resolver version;
- reviewer;
- timestamp;
- downstream impact.

An unmerge may require reindexing, cache invalidation, feature correction, model-label repair, and re-evaluation of approvals or actions.

## Temporal graph model

Use bitemporal relationships when both business truth and knowledge history matter.

Each claim can have:

- `valid_from`, `valid_to`;
- `recorded_at`, `superseded_at`;
- source revision;
- resolution state.

Example:

```text
RFI-17 REFERENCES Drawing A-201 rev 2
valid: 2026-01-10 to 2026-02-14
recorded: 2026-01-11
superseded: 2026-02-15
```

Queries:

- **current truth:** what is valid now?
- **valid-time:** what applied on January 20?
- **system-time:** what did the platform know on January 20?
- **bitemporal:** what did the platform believe on January 20 about what applied January 15?

Approval and audit queries often need system-time. A later correction must not rewrite what evidence was available to an earlier decision.

## Relationship provenance

Represent relationship evidence as a first-class object or properties.

A useful claim record includes:

- claim ID;
- subject, predicate, object;
- tenant/security scope;
- source record and version;
- extraction rule/model;
- evidence span/page/region;
- valid and system time;
- confidence;
- asserted/inferred/reviewed state;
- reviewer;
- policy/ontology version.

If multiple sources support one edge, retain them separately. Source disagreement should remain visible.

An inferred edge should point to the rule and parent claims. Do not flatten inference into an unexplained fact.

## Ingestion and synchronization

Sources may arrive through APIs, CDC, events, files, or batch snapshots.

A safe pipeline:

1. acquire source event/version;
2. validate schema and authorization scope;
3. upsert immutable source record;
4. compute candidate entities/relationships;
5. apply deterministic constraints;
6. queue ambiguous resolution;
7. write versioned graph claims;
8. update graph/index projection atomically or with publication version;
9. emit downstream change event;
10. reconcile counts and versions with the source.

Use source event ID and transformation version for idempotency.

Out-of-order events require source versions or event time. A stale delete or update must not overwrite a newer projection.

For snapshot sources, compare snapshots and tombstone removed records deliberately. Absence from one partial export is not necessarily deletion.

## Storage choices

### Property graph

Nodes, relationships, and properties.

Good for traversals and application-oriented graph models.

### RDF/triple store

Subject–predicate–object statements, shared vocabularies, and semantic standards.

Good for interoperability, formal vocabularies, and linked data.

### Relational representation

Node and edge tables with indexed keys and temporal columns.

Good when operations already run on relational infrastructure and traversal depth is bounded.

### Search/vector projection

Useful for retrieval over node text and neighborhood summaries.

Not authoritative graph storage.

Choose from query patterns, scale, transaction needs, temporal support, ecosystem, team skill, and portability. Benchmark real traversals and updates.

## Query design

Bound traversals by:

- tenant/security scope;
- relationship type;
- valid/system time;
- maximum depth;
- node/edge limit;
- source confidence/state;
- deadline and cost.

Unbounded variable-length paths can explode.

Prefer explicit question-specific paths:

```text
InvoiceLine
  -> CLAIMS_AGAINST Commitment
  <- MODIFIES ChangeOrder
  -> SUPPORTED_BY RFI
  -> REFERENCES DrawingRevision
```

Return paths with evidence, not only endpoint entities.

Use pagination and continuation tokens for large neighborhoods. Apply authorization during traversal, not after constructing the full result.

## Authorization in graphs

Graphs increase inference risk: even when a node is hidden, its relationships can reveal sensitive information.

Enforce:

- trusted tenant scope at entry;
- node and edge authorization;
- purpose and relationship constraints;
- field-level redaction;
- traversal budget;
- derived/inferred fact policy;
- query and result auditing;
- secure caches and embeddings.

A user authorized for one project cannot follow a shared vendor edge into another project unless policy explicitly permits it.

Do not rely on a final response filter. Unauthorized nodes must not influence ranking, summaries, counts, or model context.

## Graph-assisted retrieval

A graph can improve RAG:

1. resolve entities in the query;
2. traverse allowed relationships;
3. retrieve authoritative source records;
4. combine graph constraints with lexical/vector search;
5. rerank by relevance, authority, time, and provenance;
6. pass cited evidence, not unexplained graph IDs, to the model.

Benefits:

- disambiguation;
- multi-hop evidence;
- revision-aware retrieval;
- explainable paths;
- impact analysis.

Risks:

- entity-linking errors amplify across hops;
- stale edges retrieve obsolete records;
- graph structure leaks unauthorized facts;
- path explosion;
- model treats inferred edges as authoritative;
- prompt injection in source text.

Treat graph claims as data with provenance. Deterministic policy, not the LLM, controls traversal and authorization.

## Reasoning and inference

Rules can derive facts:

- if drawing revision B supersedes A, current queries should prefer B;
- if an invoice line references a commitment modified by an approved change order, surface the order as candidate support;
- if a source record resolves to an organization, its authorized invoices may join that entity.

Version rules and evaluate them. Use monotonic and non-monotonic reasoning carefully: a later fact may invalidate an earlier conclusion.

Store inferred claims separately from asserted claims. Include proof paths.

LLM-generated hypotheses can create candidates for review, not trusted graph facts.

## Graph analytics and embeddings

Graph algorithms may support:

- connected components;
- shortest paths;
- centrality;
- community detection;
- link prediction;
- anomaly detection;
- graph embeddings.

Validate whether the graph construction makes the metric meaningful. High degree can reflect data integration, not operational importance.

Graph embeddings can help candidate generation or retrieval, but they obscure provenance and may leak neighborhood information. Scope by tenant, version models, and evaluate sensitive inference.

Never let a link-prediction score establish patient, vendor, or payment identity without evidence and policy.

## Scaling and partitioning

Common partition boundaries:

- tenant;
- region/residency;
- project or facility;
- entity type;
- time;
- graph community.

Tenant partitioning simplifies isolation but makes authorized cross-tenant reference data harder. Shared reference nodes need controlled namespaces and access semantics.

Hot nodes—large vendors, standard specifications, shared providers—can dominate traversals. Use adjacency pagination, degree limits, cached authorized projections, and precomputed question-specific views.

Distributed graph traversals are expensive. Keep transactional ownership and critical invariants in authoritative stores; use the graph for connected queries and derived insight.

## Deletion and correction

A source correction can affect:

- entity match;
- canonical properties;
- relationships;
- inferred edges;
- search indexes;
- features;
- training/evaluation datasets;
- prior summaries;
- approval evidence.

Use lineage to find affected derivatives.

For deletion:

1. authorize and scope request;
2. tombstone source record;
3. remove/restrict graph claims;
4. recompute clusters and inferred edges;
5. update indexes/caches/embeddings;
6. propagate to datasets under policy;
7. verify absence and record exceptions;
8. prevent reingestion from connectors.

Deletion of one source does not always delete a canonical entity if other lawful sources support it. Preserve the decision and evidence.

## Failure handling

Plan for:

- source schema change;
- duplicate or out-of-order events;
- partial snapshot;
- entity-resolution model drift;
- cluster explosion;
- false merge;
- stale edge;
- graph store outage;
- traversal timeout;
- authorization service failure;
- hot node;
- reindex lag;
- deletion backlog;
- ontology version conflict.

Fail closed for sensitive traversal when authorization or scope cannot be established.

Keep graph projections rebuildable from durable source records, resolution events, and claim history. Back up canonical IDs and merge/split history because regenerating different identities can break downstream references.

## Construction operations example

A contractor wants to understand whether invoice line `INV-552/17` is supported and what would be affected if change order `CO-184` is revised.

Graph entities:

- tenant, project, organizations;
- source vendor records and canonical vendor;
- contract, commitment, invoice, invoice line;
- change order and approval revision;
- RFI, submittal, drawing revision, room/location;
- ERP proposal and approval.

Design:

1. ingest authoritative IDs and versions from ERP and project systems;
2. resolve vendor records using tenant-scoped IDs, tax/address/name evidence, and review;
3. preserve each source record;
4. create typed claims with evidence and valid/system time;
5. link invoice lines to commitments from authoritative keys;
6. link candidate change-order support through cited documents;
7. mark drawing supersession explicitly;
8. query as of invoice submission and current time separately;
9. authorize every hop within project scope;
10. return evidence paths and unresolved conflicts;
11. require project-controls review for ambiguous vendor or contractual matches;
12. bind any ERP proposal to the exact graph/source versions and revalidate authoritative records before execution.

A later-approved change order may support the line today but must not appear as evidence available when the original approval occurred.

If two vendor records are falsely merged, unmerge them, recompute affected paths, identify features/answers/proposals that used the cluster, and route consequential cases for repair.

## Healthcare operations variation

A healthcare operations graph may connect:

- source patient records and enterprise patient identity;
- encounters, claims, authorizations, referrals;
- providers, organizations, locations;
- FHIR resources and provenance;
- payer policy versions;
- operational tasks.

Patient identity resolution is high risk. Exact identifiers still require issuer and validity; probabilistic matches need strict thresholds and trained review. Never merge patients solely from name, date of birth, or an embedding.

Enforce organization, patient, encounter, purpose-of-use, consent, and region at every traversal. Shared provider/reference nodes must not expose another patient or organization’s relationships.

Temporal queries distinguish what was documented, what was available, and what policy applied at the operational decision. The graph can identify missing packet components, but it cannot infer medical necessity or deny care.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Knowledge graph | Flexible multi-hop queries | Ontology and operations burden |
| Relational joins | Strong transactions and familiarity | Rigid for evolving traversals |
| Property graph | Natural application traversal | Vendor/query-language variation |
| RDF/semantic model | Interoperability and reasoning | Modeling complexity |
| Canonical entity | Cross-source connection | False-merge risk |
| Keep source records | Reversible provenance | More storage/query complexity |
| Deterministic matching | Explainable precision | Low recall with dirty data |
| Probabilistic matching | Handles noise | Calibration/review burden |
| Auto-merge | Scale | Consequential identity errors |
| Human resolution | Accountability | Queue delay |
| Bitemporal edges | Honest history | Storage/query complexity |
| Inferred relationships | Useful discovery | Can be mistaken for fact |
| Graph RAG | Multi-hop grounded context | Error/leak amplification |
| Precomputed paths | Fast serving | Staleness and storage |
| Live traversal | Fresh flexibility | Latency and path explosion |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Graph becomes source of truth | Derived error overrides records | Authoritative links and revalidation |
| Collapse source records on merge | No unmerge/audit | Canonical overlay |
| Match IDs without namespace | Cross-tenant/entity collision | Issuer-scoped identifiers |
| Similar name auto-merges | Identity corruption | Multi-signal thresholds/review |
| Pairwise matches assumed transitive | Invalid cluster | Cluster constraints |
| No negative evidence | Contradictions ignored | Explicit conflict features |
| Edge lacks source/version | Unverifiable fact | Claim provenance |
| Correction overwrites history | Audit is false | Bitemporal append-only claims |
| Query uses current graph for past decision | Future leakage | As-of valid/system-time query |
| Authorization applied after traversal | Sensitive inference leak | Per-hop enforcement |
| Shared vendor crosses projects | Tenant/project exposure | Scoped relationships |
| Unbounded path query | Resource exhaustion | Typed depth/cost limits |
| LLM writes graph facts | Hallucinated truth | Candidate-only plus validation |
| Inferred edge looks asserted | Overtrust | State and proof path |
| Snapshot absence treated as delete | Data loss | Completeness/reconciliation |
| False merge not propagated | Bad answers/features persist | Lineage-driven repair |
| Deleted record reingested | Retention violation | Connector tombstone |
| Graph embedding crosses tenants | Information leakage | Scoped training/indexing |
| Hot reference node | Latency outage | Pagination/precomputation/limits |

## What to measure

Track source-to-graph reconciliation; ingestion lag; idempotent duplicate rate; entity-resolution precision/recall; auto-match, review, no-match, merge, and unmerge rates; candidate blocking recall; cluster-size anomalies; edge provenance completeness; stale and superseded edges; temporal-query correctness; unauthorized traversal attempts; query latency and expansion; hot-node load; graph-assisted retrieval quality; inferred-edge accuracy; deletion/correction propagation; affected-output discovery time; and the percentage of graph-backed answers with verifiable evidence paths.

A graph is useful when it makes connected questions more correct and explainable—not when it has the most nodes.

## Interview prompt

> Design a multi-tenant knowledge graph for a construction AI platform. It must resolve organizations across ERP, project-management systems, contracts, and documents; connect invoices, change orders, RFIs, and drawing revisions; answer historical “as of” questions; support RAG; and preserve authorization. Adapt it for healthcare patient, provider, claim, and authorization operations.

Cover ontology, identity, entity resolution, provenance, temporal modeling, ingestion, storage, traversal, graph RAG, inference, scaling, security, correction/deletion, failure recovery, and downstream action safety.

## Worked answer

**One-line conclusion:** Build the graph as a tenant-scoped, bitemporal, provenance-rich projection over immutable source records, with reversible entity resolution and authorization enforced on every traversal and action boundary.

“I would keep ERP, project-management, and document systems authoritative. Their immutable source records enter a graph projection with tenant/source IDs. Canonical entities sit above source records; merges never delete source identity and every match records evidence, confidence, resolver version, and review.

The ontology uses typed relationships such as invoice-line claims against commitment, change order modifies contract, RFI references drawing revision, and drawing supersedes drawing. Each claim includes source, evidence region, valid time, system time, asserted/inferred state, and authorization scope.

Entity resolution normalizes records, generates candidates with multiple blocking rules, scores positive and negative evidence, auto-matches only above a calibrated high threshold, routes ambiguity to experts, and enforces cluster constraints. Merge and split events are reversible and trigger downstream repair.

Ingestion is idempotent and version-aware. Snapshot deletions require completeness checks. Graph queries are bounded by tenant, type, time, depth, confidence, and cost. Authorization applies at every node and edge; hidden relationships cannot influence counts, ranking, or model context.

For RAG, I resolve query entities, traverse approved paths, retrieve current authoritative documents, and return cited paths with provenance. The LLM may propose candidate links but cannot assert facts or authorize tools.

For historical approvals I query both valid and system time, then revalidate current authoritative records before any ERP action. I monitor resolution quality, cluster anomalies, stale edges, traversal latency, authorization, deletion, and every downstream output affected by a merge correction.

Healthcare uses the same architecture with much stricter patient-identity thresholds, purpose/consent/region controls, FHIR provenance, and no automated inference of medical necessity.”

## Practice lab

Design the graph for a project where:

- ERP and Procore use overlapping vendor IDs;
- one vendor changed legal name and address;
- two companies share a phone number;
- an invoice references a commitment and three possible change orders;
- one change order was approved after invoice submission but backdated;
- an RFI references a superseded drawing;
- a source snapshot omits 15% of records;
- a resolver falsely merges two subcontractors;
- a user can access only one of their projects;
- an LLM proposes a new edge;
- the vendor later requests deletion.

Specify node/edge types, source and canonical IDs, blocking/scoring, thresholds, cluster constraints, valid/system time, provenance, per-hop authorization, bounded queries, RAG evidence, merge repair, snapshot handling, deletion, and metrics.

Then adapt it to patient/provider/claim identity with FHIR provenance, delayed records, shared addresses, purpose-of-use, and a rule that graph inference cannot establish medical necessity or merge patients without governed evidence.

## References

- [W3C RDF 1.1 Concepts and Abstract Syntax](https://www.w3.org/TR/rdf11-concepts/)
- [W3C PROV-O: The PROV Ontology](https://www.w3.org/TR/prov-o/)
- [W3C Time Ontology in OWL](https://www.w3.org/TR/owl-time/)
- [A Theory for Record Linkage — Fellegi and Sunter](https://doi.org/10.1080/01621459.1969.10501049)
- [Apache TinkerPop documentation](https://tinkerpop.apache.org/docs/current/reference/)
- [HL7 FHIR Patient](https://hl7.org/fhir/R4/patient.html)
- [HL7 FHIR Provenance](https://hl7.org/fhir/R4/provenance.html)
