# Multimodal document intelligence, OCR, layout, and evidence extraction

Document intelligence converts files and images into structured, citable evidence. OCR is one stage—not the whole system.

A production pipeline must preserve what the source said, where it appeared, which transformations were applied, how certain the extraction is, and when a human must review it.

## Mental model

Treat a document as a **versioned spatial evidence graph**, not a bag of text.

The source contains multiple coordinated signals:

- characters and handwriting;
- bounding boxes and reading order;
- headings, paragraphs, lists, and footnotes;
- tables, cells, rows, and merged regions;
- checkboxes, signatures, stamps, and annotations;
- diagrams, symbols, dimensions, and legends;
- embedded images and metadata;
- relationships across pages and revisions.

The pipeline creates three layers:

1. **source layer** — immutable original bytes, page renderings, file identity, and access scope;
2. **observation layer** — OCR tokens, coordinates, layout regions, table cells, visual detections, and confidence;
3. **interpretation layer** — normalized fields, entities, relationships, evidence claims, and business validation.

Never overwrite observation with interpretation. A normalized invoice total can point to an OCR token and page region while preserving the original image and raw OCR.

The core invariant is:

> Every extracted value can be traced to a specific source version and visual region, with transformation versions and uncertainty attached.

## When to use it

Use document intelligence when systems must process:

- scanned PDFs, faxes, photographs, or handwriting;
- invoices, pay applications, contracts, RFIs, submittals, and change orders;
- drawings, schedules, legends, and marked-up plans;
- forms with checkboxes, signatures, or stamps;
- tables whose meaning depends on rows and columns;
- mixed text and images;
- medical claims, referrals, prior authorizations, or clinical forms;
- documents requiring citations, audit, or human verification.

Prefer deterministic parsers for reliable born-digital structure such as valid XML, JSON, CSV, or known PDF form fields. Do not rasterize good structured data merely to feed it through OCR.

Use human entry or assisted review when the source quality, consequence, or document variability makes automatic extraction economically or operationally unsafe.

## Processing architecture

A durable pipeline usually contains:

1. upload or connector acquisition;
2. malware and format validation;
3. immutable source storage;
4. document classification and routing;
5. page rendering and image preprocessing;
6. OCR and handwriting recognition;
7. layout and reading-order analysis;
8. table, key-value, checkbox, signature, and visual extraction;
9. cross-page assembly;
10. normalization and business validation;
11. confidence calibration and review routing;
12. publication to search, workflow, and analytical stores.

Each stage is versioned, idempotent, retryable, and observable. Store stage outputs separately so a new table model does not require repeating malware scanning or source acquisition.

Use a document ID plus source-content digest and pipeline version as the idempotency boundary. A new source revision or pipeline bundle creates a new derived version; it does not mutate prior evidence silently.

## Classify before extracting

One pipeline should not treat every page identically. Route by properties such as:

- born-digital text versus raster scan;
- form, prose, table, drawing, photograph, or mixed page;
- typed versus handwritten content;
- language and script;
- orientation, skew, blur, contrast, and resolution;
- page size and density;
- expected template or vendor family;
- sensitivity and allowed processing region.

Classification can be hierarchical: document type, page type, then region type.

Keep a safe fallback. If classification confidence is low, run a general parser or send the document to review; do not confidently apply the wrong vendor template.

## Preprocessing

Image preprocessing may include:

- orientation detection and rotation;
- deskewing;
- dewarping photographed pages;
- denoising;
- contrast and illumination correction;
- border and punch-hole removal;
- page segmentation;
- resolution normalization;
- color preservation for annotations;
- perspective correction.

Every transformation can also destroy evidence. Aggressive thresholding may erase light pencil marks, colored revision clouds, stamps, or faint signatures.

Preserve the original rendering, record preprocessing parameters, and evaluate preprocessing by downstream task quality—not visual cleanliness alone. Generate multiple representations when different tasks need them, such as a high-contrast OCR image and a color image for annotations.

## OCR output

Useful OCR output includes more than text:

- token, line, and paragraph text;
- page and polygon/bounding-box coordinates;
- reading-order candidates;
- language;
- confidence or alternative characters;
- font/style cues when available;
- engine and model version;
- preprocessing version;
- links to the source page.

Raw OCR confidence is not automatically a probability that a business field is correct. Calibrate confidence on representative data and at the level that matters: character, token, field, row, document, or decision.

Common confusions include:

- `0/O`, `1/I/l`, `5/S`, and punctuation;
- decimal points and thousands separators;
- negative values in parentheses;
- currency symbols;
- dates across locales;
- unit abbreviations;
- handwritten digits;
- superscripts and drawing callouts.

Use domain validation as evidence, not as permission to invent. If subtotal plus tax does not equal total, surface the conflict; do not silently replace the observed total.

## Layout and reading order

Text order in a PDF object stream may not match human reading order. Multi-column pages, sidebars, headers, footers, notes, and callouts require spatial reasoning.

Represent layout as regions and relationships:

- contains;
- precedes;
- labels;
- value-of;
- row-of and column-of;
- caption-of;
- references;
- continues-on-next-page.

Reading order may be task-specific. A contract reader may follow paragraphs; a form extractor connects labels to nearby values; a drawing reader connects a keynote symbol to a legend and plan location.

Keep coordinates in a canonical page space and record any transformation between original, rendered, and model image coordinates. This allows reliable citation overlays and review interfaces.

## Tables

Tables are structures, not tab-separated OCR text.

The system must infer:

- table boundary;
- row and column lines, including borderless tables;
- header levels;
- merged and spanning cells;
- row continuation across pages;
- units and footnotes;
- empty versus missed cells;
- cell reading order.

Store cells with row/column spans, coordinates, raw text, normalized value, confidence, and header path.

Validate tables with domain rules. For an invoice:

- line totals should reconcile with quantity and unit price where applicable;
- subtotal, tax, retainage, and total should reconcile;
- currency and sign conventions should be consistent;
- duplicate line or page detection should run;
- continuation pages should not repeat headers as data.

Validation failure should lower decision confidence or trigger review, not cause an unsupported correction.

## Key-value fields and entities

Field extraction combines multiple signals:

- OCR text;
- spatial proximity;
- layout region;
- template or anchor;
- semantic model;
- business vocabulary;
- cross-field constraints;
- external master data.

Separate three values:

1. **observed value** — literal source/OCR output;
2. **normalized value** — canonical date, money, unit, vendor, or code;
3. **resolved entity** — link to an authoritative vendor, project, patient, or contract.

For example, “Rodríguez Const.” may normalize as text but should link to a vendor master only when matching evidence is sufficient. Record the matching method and confidence; ambiguous entities go to review.

## Drawings and visual evidence

Construction drawings require more than OCR. Useful components include:

- title-block and revision-table extraction;
- sheet number, discipline, scale, issue date, and status;
- symbol and keynote detection;
- legend association;
- room, grid, and level references;
- dimension and measurement extraction;
- detail and section-callout links;
- revision-cloud and markup detection;
- alignment across sheet revisions.

Do not infer real-world distance from pixels unless scale, viewport, and calibration are valid. A scanned or resized drawing can invalidate nominal scale.

Preserve vector geometry when available. Rasterize only for models that need images, and retain the coordinate transform back to the source.

A vision-language model can explain a region or connect text and graphics, but its output remains a proposed interpretation. Require region citations and deterministic checks for sheet identity, revision, units, and consequential quantities.

## Multimodal models

Multimodal models are useful for:

- document and page classification;
- flexible field extraction;
- answering questions about charts or diagrams;
- associating labels with visual regions;
- handling variable templates;
- generating candidate summaries.

They introduce risks:

- hallucinated text or visual objects;
- weak coordinate precision;
- sensitivity to image resizing and tiling;
- inconsistent structured output;
- hidden OCR behavior;
- high latency and cost;
- prompt injection embedded in images;
- privacy and residency restrictions.

Use schema-constrained outputs, source-region references, task-specific evaluation, and deterministic validation. Treat text read from images as untrusted document content, never control instructions.

For large pages, use layout-aware tiling with overlap and a page-level reconciliation step. Record tile coordinates and transforms so extracted evidence maps back to the original.

## Evidence model and provenance

An evidence object can contain:

```json
{
  "evidence_id": "ev_...",
  "tenant_id": "t_...",
  "document_id": "doc_...",
  "document_version": 7,
  "source_digest": "sha256:...",
  "page": 12,
  "region": {"polygon": [[0.12,0.31],[0.48,0.31],[0.48,0.37],[0.12,0.37]]},
  "raw_text": "Total Due 18,420.00",
  "normalized_value": {"amount": "18420.00", "currency": "USD"},
  "field_type": "invoice_total",
  "confidence": 0.94,
  "extractor_version": "invoice-bundle/23",
  "validation": ["subtotal_plus_tax_matches"],
  "review_status": "unreviewed"
}
```

Normalized coordinates simplify display across resolutions, but keep the exact page-render version and transform.

A citation must resolve to the current authorized source. If the source is deleted, superseded, or access is revoked, derived evidence and search entries need tombstones and propagation.

## Confidence and uncertainty

Combine signals deliberately:

- OCR confidence;
- detector or extractor score;
- calibration result;
- cross-field validation;
- agreement across models or passes;
- source quality;
- template/classifier confidence;
- entity-resolution margin;
- novelty or out-of-distribution signal.

Avoid averaging unrelated scores into a misleading number. A document can have high OCR confidence but an invalid table structure.

Define action thresholds by consequence:

- low-risk indexing may accept lower confidence;
- an invoice payment field needs stronger evidence;
- a clinical or coverage decision may require mandatory human review;
- missing or conflicting critical fields should abstain regardless of average score.

Calibrate on representative slices and monitor calibration drift. A score of 0.9 should have measured meaning for the defined task and population.

## Human review

A review interface should show:

- source page and highlighted region;
- raw OCR and proposed normalized value;
- related fields and reconciliation checks;
- model/version and uncertainty reason;
- previous revisions when relevant;
- keyboard-efficient correction;
- reviewer identity and timestamp;
- escalation and “cannot determine” options.

Do not ask reviewers to confirm hundreds of high-confidence fields merely to create the appearance of control. Route by risk, uncertainty, novelty, validation failures, and sampling needs.

Reviewer corrections are valuable labels, but they are not automatically ground truth. Preserve original proposal and correction, capture reason codes, audit disagreement, and adjudicate consequential examples before training.

## Publication and downstream use

Publish a document version atomically only when its required stages complete. Search and workflow consumers should not combine fields from different pipeline versions accidentally.

Keep:

- source version;
- extraction bundle version;
- publication state;
- evidence IDs;
- review status;
- supersession and deletion state.

Derived search chunks should carry page/region provenance and authorization metadata. Structured fields used for actions must declare whether they are machine-extracted, human-verified, or sourced from an authoritative system.

A later correction can invalidate summaries, embeddings, approvals, or proposed actions. Emit a versioned change event and let consumers revalidate their dependencies.

## Failure handling

Plan for:

- corrupt or encrypted PDFs;
- unsupported fonts or malformed objects;
- decompression bombs and huge pages;
- missing pages;
- rotated, blurred, or shadowed images;
- OCR provider timeout;
- duplicate queue delivery;
- partial multipage results;
- page-order mistakes;
- table continuation failure;
- mismatched coordinate transforms;
- model output that violates schema;
- stale document revision;
- deleted source with lingering derivatives;
- reviewer backlog;
- regional provider outage.

Persist stage state and retry only failed idempotent work. A document should never appear fully processed when required pages are missing.

Bound files by bytes, pages, pixels, render time, and model tokens. Quarantine suspicious inputs. Use dead-letter and repair workflows with visible reason codes.

## Construction operations example

A contractor receives a 48-page subcontractor pay application containing:

- cover sheet and notarized signature;
- schedule of values;
- continuation tables;
- retainage;
- stored-material photos;
- lien waiver;
- handwritten correction;
- change-order references;
- a superseded drawing excerpt.

Design:

1. store and hash the original under project authorization;
2. classify pages into forms, tables, signatures, photos, and drawing excerpts;
3. render safely and create OCR/color variants;
4. extract title fields, table structure, signatures, and annotations with coordinates;
5. join continuation rows across pages without duplicating headers;
6. normalize money while preserving literal observations;
7. reconcile original contract, approved changes, completed work, retainage, prior payments, and current amount;
8. resolve change-order IDs against authoritative project records;
9. flag the handwritten amount and superseded drawing for review;
10. show reviewers highlighted evidence and reconciliation differences;
11. publish verified fields with source and review provenance;
12. bind any payment approval to exact document, extraction, policy, and ERP-operation versions.

A detected signature proves only that a signature-like mark exists in a region. It does not establish signer identity or legal validity without the appropriate verification process.

## Healthcare operations variation

A prior-authorization workflow receives faxed referrals, insurance cards, clinical notes, and a handwritten form.

The pipeline:

- enforces organization, patient, encounter, purpose-of-use, and region before processing;
- detects page type and missing pages;
- extracts identifiers, dates, codes, checkbox states, and clinical attachments;
- maps fields to FHIR resources without overwriting raw evidence;
- validates identifiers and terminology against approved systems;
- redacts or minimizes PHI for components that do not require it;
- routes illegible, conflicting, or clinically consequential fields to trained staff;
- records every source region and reviewer correction;
- prevents unsupported models, logs, and caches from receiving PHI.

OCR confidence does not determine medical truth. Signed clinical documentation and authoritative records have defined roles; the system surfaces conflicts rather than choosing the most fluent interpretation.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| General model | Handles variable layouts | Cost and inconsistent edge cases |
| Template extractor | High precision on stable forms | Breaks on new versions |
| OCR-first pipeline | Modular and searchable | OCR errors propagate |
| End-to-end multimodal extraction | Flexible joint reasoning | Harder provenance and calibration |
| High-resolution rendering | Better small text | Memory, latency, and cost |
| Aggressive preprocessing | Cleaner OCR input | May erase evidence |
| Preserve color | Keeps markups and stamps | More storage/compute |
| Multiple model passes | Agreement and recall | Cost and correlated errors |
| Strict review threshold | Lower consequential error | Reviewer backlog |
| Broad automation | Throughput | Silent business risk |
| Store all intermediate images | Reproducibility | Privacy and storage burden |
| Recompute derivatives | Simpler deletion | Recovery latency |
| Vector preservation | Accurate geometry | Parser complexity |
| Raster-only processing | Uniform pipeline | Loses structure and scale fidelity |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Treat document as plain text | Loses table/layout meaning | Spatial evidence graph |
| Rasterize born-digital structure | Lower accuracy | Parse native structure first |
| Discard coordinates | Citations cannot be verified | Preserve canonical regions |
| Trust PDF text order | Scrambled reading | Layout-aware ordering |
| Flatten table to lines | Wrong row/column values | Cell and header structure |
| Normalize without raw value | Audit cannot reconstruct | Store observed and normalized |
| Auto-fix failed arithmetic | Invented evidence | Surface conflict and review |
| Trust OCR confidence as truth | Consequential field error | Calibrated field/decision confidence |
| Apply one template globally | Silent mis-extraction | Classification and novelty routing |
| Downsample drawing too far | Missed dimensions/notes | Tiling and task resolution |
| Use pixel distance as measurement | Wrong quantity | Validate scale and geometry |
| Lose tile transform | Bad citations | Coordinate lineage |
| VLM follows image instruction | Injection/tool misuse | Treat pixels as untrusted evidence |
| Publish partial pages | Incomplete business record | Atomic completeness checks |
| Retry non-idempotently | Duplicate derivatives/review tasks | Source+pipeline idempotency |
| Mix pipeline versions | Internally inconsistent result | Versioned atomic publication |
| Reviewer correction becomes truth | Label contamination | Provenance and adjudication |
| Deleted source remains in index | Privacy/retention violation | Tombstones and verified propagation |
| Backlog hidden by auto-accept | Unsafe automation | Queue SLOs and graceful degradation |

## What to measure

Track document and page throughput; render/OCR/extraction latency; cost per page and verified field; unsupported/quarantined files; missing-page detection; character and word error rate where relevant; field precision, recall, and exact match; table detection and cell-structure quality; normalized-value accuracy; citation-region accuracy; entity-resolution accuracy; calibration error; abstention; review rate and queue age; reviewer agreement and correction reasons; duplicate publication; pipeline-version mix; deletion propagation; quality by document type, vendor, scan quality, handwriting, language, page count, and consequence.

Measure end-to-end business correctness. Low character error rate can coexist with a wrong invoice total or mismatched table row.

## Interview prompt

> Design a multi-tenant document-intelligence platform for construction operations. It processes invoices, pay applications, contracts, RFIs, drawings, photos, and handwritten markups; produces cited structured evidence; and proposes ERP actions. Adapt the design for faxed healthcare authorization documents containing PHI.

Cover source storage, classification, preprocessing, OCR, layout, tables, drawings, multimodal models, provenance, confidence, review, publication, scaling, failures, security, evaluation, and downstream correction.

## Worked answer

**One-line conclusion:** Build a versioned spatial-evidence pipeline that preserves source regions and raw observations, validates structured interpretations, and sends consequential uncertainty to accountable human review before downstream action.

“I would acquire each file into quarantined immutable storage, validate it, compute a digest, and enforce tenant/project authorization. A classifier routes born-digital, scanned, form, table, drawing, photo, and handwriting pages to appropriate parsers.

Page rendering and preprocessing are versioned and preserve originals. OCR returns text, coordinates, language, and confidence. Layout analysis creates regions and reading-order relationships; table extraction keeps cells, spans, headers, continuations, and units. Drawings preserve vector geometry when possible and connect title blocks, revisions, symbols, legends, and callouts. Multimodal models propose schema-constrained interpretations with cited regions but never authorize actions.

Every field stores raw observation, normalized value, source digest, page polygon, pipeline version, validation results, and review status. Business rules reconcile invoice arithmetic, contract and change-order versions, duplicate documents, and authoritative entity IDs. Confidence is calibrated by field and risk slice; conflicting or consequential fields abstain into a review queue with highlighted evidence.

Publication is atomic for a document/pipeline version. Search chunks and structured records inherit authorization and provenance. Corrections or deletions emit versioned events that invalidate summaries, indexes, approvals, and action proposals.

Workers are idempotent by source digest and pipeline bundle, queues are bounded by pages/pixels, and partial stages resume without publishing incomplete documents. I would evaluate field and table accuracy, citation coordinates, calibration, review load, document slices, injection, cost, and the final business decision. Healthcare adds PHI purpose and region controls, FHIR mapping provenance, stricter minimization, and trained review for clinical or coverage uncertainty.”

## Practice lab

Design the pipeline for a 48-page pay application with:

- a born-digital cover sheet;
- low-resolution scanned continuation tables;
- merged cells and repeated headers;
- a handwritten tax correction;
- stored-material photos;
- a signature and notary stamp;
- change-order references;
- a superseded drawing excerpt;
- one missing page;
- a prompt-injection instruction embedded in an image;
- an OCR timeout after 31 pages;
- a later source deletion request.

Specify routing, image variants, OCR/layout/table outputs, evidence schema, confidence and review rules, idempotency, atomic publication, correction/deletion propagation, evaluation slices, and how an ERP proposal is bound to verified evidence.

Then adapt the design to a faxed prior-authorization packet with PHI, checkboxes, handwriting, clinical notes, and conflicting patient identifiers.

## References

- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [LayoutLMv3: Pre-training for Document AI with Unified Text and Image Masking](https://arxiv.org/abs/2204.08387)
- [DocVQA: A Dataset for VQA on Document Images](https://arxiv.org/abs/2007.00398)
- [FUNSD: A Dataset for Form Understanding in Noisy Scanned Documents](https://arxiv.org/abs/1905.13538)
- [PubTables-1M: Towards Comprehensive Table Extraction From Unstructured Documents](https://arxiv.org/abs/2110.00061)
- [W3C Web Annotation Data Model](https://www.w3.org/TR/annotation-model/)
- [HL7 FHIR DocumentReference](https://hl7.org/fhir/R4/documentreference.html)
