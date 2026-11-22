# Training-data deduplication, contamination detection, and benchmark integrity

A training dataset can contain millions of rows yet represent far fewer independent examples. Files are copied, templates are filled repeatedly, documents are re-exported, web pages mirror each other, and the same project evidence appears in email, storage, and an ERP.

Duplicates distort sampling, encourage memorization, leak across evaluation splits, overweight prolific sources, waste compute, and create false confidence. Benchmark contamination is especially dangerous: a model can reproduce answers it encountered during training without learning the intended capability.

## Mental model: govern example families, not isolated rows

Build a provenance and similarity graph:

```text
source record
 -> immutable raw object + rights/purpose
 -> canonical representation
 -> exact and near-duplicate edges
 -> semantic / entity / temporal family
 -> split and sampling policy
 -> immutable dataset manifest
 -> training and evaluation lineage
```

Distinguish four relationships:

1. **Exact duplicate:** identical canonical bytes or content.
2. **Near duplicate:** small formatting, OCR, crop, encoding, or wording changes.
3. **Related family:** same underlying entity, event, template, project, patient, contract, or answer.
4. **Independent example:** sufficiently separate for the evaluation claim.

Hash equality can prove exact sameness under one canonicalization. It cannot prove independence.

## When to deduplicate

Deduplicate or group data when:

- sources are crawled, exported, mirrored, or joined from multiple systems;
- templates create many superficially different examples;
- train, validation, and test splits share entities or time windows;
- repeated examples dominate token or gradient mass;
- evaluation material may be publicly available;
- synthetic or model-generated data rephrases existing examples;
- privacy, licensing, or deletion must propagate to derived copies.

Do not blindly delete every repeated record. Frequency may carry real information: a recurring equipment alarm or frequently used phrase can reflect production prevalence. Preserve raw lineage, then choose whether to collapse, cap, group, or reweight repetitions for a stated objective.

## Define the unit and evaluation claim

Before selecting an algorithm, define what must be independent.

| Evaluation claim | Grouping boundary |
|---|---|
| Generalize to unseen invoice pages | Document or invoice family |
| Generalize to new vendors | Vendor/entity |
| Generalize to new projects | Project and organization |
| Forecast future outcomes | Strict time boundary |
| Handle new drawing revisions | Drawing lineage, with prior revisions excluded as needed |
| Generalize to new facilities | Facility or hospital system |
| Detect unseen templates | Template or layout family |
| Answer novel benchmark questions | Prompt, solution, paraphrase, and source family |

Random row splitting is usually wrong when rows share an entity, document, event, or template.

## Immutable ingestion and provenance

Retain an immutable record for every source item:

- source system, URI or object identity, and collection time;
- content hash and byte size;
- tenant, purpose, consent, license, and retention;
- source revision and parent/child relationship;
- parser, OCR, normalization, and language versions;
- deletion and legal-hold state;
- trust level and ingestion principal.

Derived text, image crops, chunks, embeddings, and synthetic variants should point back to source identities. This makes it possible to explain why an item was included, propagate deletion, and rebuild similarity groups when canonicalization improves.

## Canonicalization

Canonicalization removes irrelevant differences while preserving meaning needed for the task.

Text candidates may normalize:

- Unicode form;
- line endings and repeated whitespace;
- markup and boilerplate;
- case, only when case is not meaningful;
- stable page headers or footers;
- OCR artifacts under a versioned rule.

Document candidates may separate text, layout, tables, images, and metadata. Image candidates may normalize encoding and orientation while retaining original pixels.

Over-normalization creates false duplicates. Removing numbers could collapse distinct invoice amounts; removing room labels could merge different construction conditions. Store the canonicalizer version and keep original content.

## Exact deduplication

Hash canonical content with a collision-resistant digest. Use streaming or partitioned aggregation for scale:

1. compute canonical content and hash;
2. partition records by hash prefix;
3. group equal hashes;
4. compare canonical length or bytes where assurance requires it;
5. select a representative according to rights, quality, and provenance;
6. retain all lineage edges.

Do not choose the first record blindly. Prefer the source with approved usage, higher extraction quality, stable identity, and complete metadata.

Exact deduplication is deterministic and cheap, but misses small edits, crops, OCR differences, and paraphrases.

## Near-duplicate detection

### Text fingerprints

Use token or character shingles, then estimate set similarity with MinHash. Locality-sensitive hashing can retrieve candidate pairs without comparing every document.

For shingle sets (A) and (B):

[
J(A,B)=rac{|A cap B|}{|A cup B|}
]

MinHash estimates Jaccard similarity. Thresholds depend on document length and task. Short documents need special handling because a few shared tokens produce unstable similarity.

SimHash is useful for cosine-like fingerprinting of weighted features. Suffix arrays or n-gram indexes help detect long copied spans.

### Images and documents

Use perceptual hashes for resized or recompressed images, OCR/text fingerprints for document content, and layout features for templated pages. Crops and embedded images require region-level matching.

### Embedding similarity

Embeddings can discover semantic relatives and paraphrases, but semantic similarity is not duplication. Two independent safety incidents may describe the same concept and should not be collapsed. Use embeddings to propose candidates, then apply task-aware rules or review.

## Connected components and transitivity

Pairwise similarity edges form groups. Be careful with transitivity:

- A resembles B.
- B resembles C.
- A may not resemble C.

A single-link connected component can chain unrelated records into one giant cluster. Consider complete-link constraints, representative-based assignment, maximum-diameter rules, or hierarchical clustering.

Record edge reason and score. A group should be auditable rather than an opaque cluster ID.

## Sampling and weighting

Options for duplicate families:

- keep one representative;
- cap members per family;
- sample families first, then a member;
- weight each record by inverse family size;
- preserve frequency for production-calibrated training;
- retain separate views for memorization analysis.

Deduplication changes the distribution. Report before/after counts by source, language, tenant, template, label, and cohort. A global threshold can disproportionately remove minority-language or small-tenant data because repeated templates may be their main representation.

## Leakage-safe splitting

Assign whole groups to one split. Grouping keys may include:

- entity, vendor, project, patient, facility, or user;
- document lineage and attachment hash;
- template and source family;
- event and downstream outcome;
- synthetic parent;
- time window.

Apply time splits after accounting for late-arriving records and backfills. If a future record is a revised copy of a past document, decide whether the evaluation measures revision handling or truly unseen content.

Split assignment should be deterministic from a versioned group identity and seed. Publish a manifest, not only a query.

## Benchmark contamination

Contamination occurs when evaluation questions, answers, solutions, or close variants influence training or tuning.

Search across:

- exact prompt and answer hashes;
- normalized strings;
- distinctive n-grams and long substrings;
- MinHash/LSH candidates;
- semantic paraphrases;
- source documents and solution explanations;
- translated or reformatted variants;
- synthetic data whose parent is a benchmark item;
- prompts sent to teachers or labeling models.

Checking only the question text is insufficient. A model may have seen the answer key, source article, code solution, or a paraphrased walkthrough.

Classify findings:

- exact exposure;
- near duplicate;
- same source or answer family;
- likely independent overlap;
- uncertain, requiring review.

Do not silently remove suspicious items and report the original score. Rebuild the affected training set or exclude contaminated evaluation items under a declared policy, then report both scope and limitations.

## Protected evaluation workflow

Use a dedicated evaluation boundary:

- restrict benchmark content and answer keys;
- issue opaque item IDs to training systems;
- prevent evaluation data from entering prompt logs, tuning stores, replay buffers, or support exports;
- track every access and derivative;
- quarantine model-generated explanations of protected items;
- rotate or replenish tests when repeated use causes overfitting;
- keep a final untouched holdout for major decisions.

Secrecy alone is fragile. Public benchmarks can still be useful if contamination is measured and claims are limited. Pair them with private, recent, domain-specific, or procedurally generated evaluations.

Use canary strings or honey items cautiously to detect data-flow violations, but do not treat absence of a canary as proof of no contamination.

## Data poisoning and source trust

Deduplication and contamination pipelines also expose attack surfaces:

- an attacker floods a phrase so it dominates sampling;
- poisoned variants avoid exact hashes;
- hidden instructions persist through OCR;
- a source copies trusted documents and inserts a backdoor;
- synthetic data recursively amplifies model mistakes;
- adversarial examples bridge similarity clusters.

Assign source trust and influence caps. Scan content before and after parsing. Maintain quarantine, review, and source-revocation workflows. A trusted URL is not proof of trusted content.

When a source is revoked, lineage should identify raw objects, canonical records, chunks, embeddings, training manifests, checkpoints, and released models affected.

## Memorization and privacy

Deduplication can reduce repeated exposure that encourages memorization, but it does not guarantee privacy.

Evaluate memorization with approved methods such as canary exposure, membership-risk analysis, rare-string reproduction tests, and targeted red-team prompts. Avoid releasing sensitive examples in reports.

Combine data minimization, access controls, purpose limitation, retention, differential privacy where appropriate, and output defenses. A unique sensitive record is not protected merely because duplicates were removed.

## Dataset release gates

A dataset manifest should include:

- source snapshots and provenance;
- canonicalizer and parser versions;
- exact and near-duplicate algorithms;
- thresholds and grouping rules;
- representative and weighting policy;
- split algorithm and seed;
- contamination scan corpus and versions;
- rights, purpose, deletion, and geographic constraints;
- before/after cohort statistics;
- reviewed exceptions;
- known limitations and unresolved uncertain matches.

Run gates for:

- zero forbidden cross-split groups;
- benchmark contamination within declared policy;
- no unauthorized source or revoked record;
- cohort representation bounds;
- label and schema validity;
- reproducible manifest hashes;
- deletion propagation;
- poisoning and anomalous-source checks.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Exact hashing | Fast, deterministic, explainable | Misses small edits and paraphrases |
| Aggressive near-dedup | Less memorization and leakage | Removes valid diversity |
| Conservative threshold | Preserves independent examples | Leaves duplicate influence |
| One representative per group | Simple, efficient | Loses real production frequency |
| Inverse-family weighting | Retains diversity with bounded influence | More complex sampling |
| Entity-level split | Strong generalization test | Fewer effective samples and distribution shift |
| Public benchmark | Comparable and inexpensive | High contamination and tuning exposure |
| Private rotating benchmark | Lower exposure | Governance, labeling, and comparability cost |
| Embedding similarity | Finds semantic variants | False positives and model-dependent bias |
| Human adjudication | Domain-aware decisions | Expensive and capacity-limited |

## Failure modes and defenses

| Failure mode | Defense |
|---|---|
| Same invoice appears as PDF, email text, and OCR output | Cross-modal/source lineage plus canonical and near-duplicate matching |
| Random split leaks vendor templates | Group by vendor, template, project, and document family |
| Removing numbers merges distinct invoices | Task-aware canonicalization and preserved originals |
| LSH cluster chains unrelated records | Diameter or representative constraints and edge audits |
| Dedup removes most small-tenant data | Cohort impact reports and balanced sampling policy |
| Benchmark question removed but solution remains | Scan prompts, answers, explanations, sources, and paraphrases |
| Synthetic paraphrase bypasses exact checks | Track synthetic parent and semantic candidate search |
| Evaluation answers leak through logs | Protected evaluation boundary and restricted derivatives |
| Attacker floods poisoned variants | Source trust, family influence caps, anomaly detection, quarantine |
| Deleted record remains in a checkpoint | End-to-end dataset-to-model lineage and retraining/unlearning policy |
| Deduplicated score is reported without rescan | Versioned contamination scan for every dataset/model release |
| Absence of matches is claimed as proof | Report methods, coverage, thresholds, and residual uncertainty |
| Training improves because test items leaked | Clean holdout, family-safe split, and exposure-aware score reporting |

## Construction example: invoice and RFI dataset

A construction company builds a model from invoices, RFIs, drawing sheets, submittals, email attachments, and exports from multiple project systems.

The same invoice exists as an original PDF, an email attachment, OCR text, and a corrected export. Exact hashes catch repeated attachments, while text shingles and layout fingerprints connect OCR and re-export variants. The family retains all provenance but receives bounded training weight.

RFIs frequently reuse company templates. Random splitting would put nearly identical RFIs from the same project into train and test. The split therefore groups by project, RFI lineage, subcontractor, and template depending on the evaluation claim.

A public construction-question benchmark has been used in prior prompt experiments. The contamination scan checks questions, answers, source manuals, generated explanations, and paraphrases stored in prompt logs. Exposed items are excluded from the clean score, and a recent private holdout is used for release.

A healthcare-operations dataset requires stronger patient, encounter, facility, and time grouping. De-identification does not make two records independent, and duplicate detection must not create an unauthorized cross-facility linkage service.

## Interview prompt

**Design a training-data integrity platform for a construction AI system ingesting invoices, drawings, RFIs, emails, OCR, and synthetic examples. Prevent duplicate over-weighting, entity leakage, benchmark contamination, poisoning, and deletion failures while preserving useful frequency and auditability.**

## Worked answer

1. **Clarify claims and boundaries.** Define tasks, independence unit, source rights, tenants, sensitive data, evaluation claims, deletion duties, and adversary model.
2. **Preserve raw provenance.** Ingest immutable objects with source, revision, purpose, rights, collection time, trust, and parent/derivative lineage.
3. **Canonicalize by modality.** Version text, document, image, layout, and OCR normalization without discarding originals or task-critical fields.
4. **Detect exact and near duplicates.** Use cryptographic hashes, shingling plus MinHash/LSH, perceptual hashes, layout/text signals, and embeddings only for candidate discovery.
5. **Build auditable families.** Store edge reasons and scores, constrain transitive clustering, select lawful high-quality representatives, and cap or weight family influence.
6. **Split by the real unit.** Assign document, entity, project, template, synthetic-parent, and temporal families atomically using deterministic manifests.
7. **Scan contamination broadly.** Search questions, answers, solutions, sources, translations, paraphrases, prompt logs, and synthetic descendants; classify exact, near, related, and uncertain exposure.
8. **Protect evaluation.** Restrict answer keys and derivatives, isolate logs and tuning stores, maintain recent private holdouts, and report exposure-aware scores.
9. **Defend and govern.** Apply source trust, influence caps, anomaly detection, quarantine, cohort-impact gates, deletion propagation, and dataset-to-model lineage.
10. **Verify releases.** Reproduce manifests, assert no forbidden cross-split families, review uncertain high-impact matches, compare cohort distributions, and record residual limitations.

**One-line conclusion:** Trustworthy training data treats duplicates and contamination as lineage-and-independence problems, grouping related evidence before splitting and reporting only scores whose exposure and provenance are understood.

## References

- [Deduplicating Training Data Makes Language Models Better](https://aclanthology.org/2022.acl-long.577/)
- [On the Resemblance and Containment of Documents](https://ieeexplore.ieee.org/document/666900)
- [Data Cards: Purposeful and Transparent Dataset Documentation for Responsible AI](https://dl.acm.org/doi/10.1145/3531146.3533231)
- [Documenting Large Webtext Corpora: A Case Study on the Colossal Clean Crawled Corpus](https://aclanthology.org/2021.emnlp-main.98/)
- [The Pile: An 800GB Dataset of Diverse Text for Language Modeling](https://arxiv.org/abs/2101.00027)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
