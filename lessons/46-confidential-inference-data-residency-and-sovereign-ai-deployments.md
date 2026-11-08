# Confidential inference, data residency, and sovereign AI deployments

AI systems often process contracts, drawings, payroll data, operational telemetry, or protected health information. Encryption at rest and in transit is necessary, but it does not protect plaintext while a CPU or accelerator is using it.

Confidential inference adds controls for **data in use**. Data residency constrains where data is stored and processed. Sovereignty goes further: it asks which legal entities, administrators, networks, keys, software supply chains, and jurisdictions can control or compel access.

These are related but different claims. A workload can run in the correct country and still expose plaintext to a privileged operator. A confidential-computing enclave can protect memory while sending logs or backups to an unapproved region.

## Mental model

Start with five questions:

1. **What must remain confidential?** Prompts, documents, embeddings, model weights, outputs, telemetry, keys, or identities?
2. **From whom?** Other tenants, cloud operators, host administrators, model providers, attackers, insiders, or a foreign legal entity?
3. **Where may each artifact exist?** Ingest, storage, processing, logs, backups, support systems, and disaster recovery.
4. **Who controls execution and keys?** Which organization can deploy code, approve updates, release keys, debug incidents, or recover data?
5. **What evidence proves the claim?** Configuration records, attestation, key-release logs, network policy, audit, deletion verification, and operational tests.

Use three separate policy dimensions:

- **location:** approved countries, regions, facilities, and recovery sites;
- **protection:** encryption, isolation, trusted execution environment, and key custody;
- **control:** personnel, organization, jurisdiction, software approval, support, and administration.

A useful authorization rule is:

[
Eligible = Location land Protection land Control land Purpose land Capability
]

All must pass before data is released. “Encrypted” or “hosted locally” is not enough.

## When to use it

Use confidential or sovereign patterns when:

- customer contracts restrict processing locations or operators;
- regulated or highly sensitive records are sent to inference;
- a tenant does not trust the infrastructure administrator with plaintext;
- organizations collaborate without exposing raw inputs to each other;
- proprietary model weights need protection from the host;
- a public model endpoint cannot satisfy retention, key, network, or audit requirements;
- government, healthcare, critical infrastructure, or defense-adjacent workloads require local operational control;
- data export, backup, or support access must remain within a jurisdiction.

Do not deploy a trusted execution environment merely as a security label. If the threat model is ordinary tenant isolation and the platform already satisfies it, confidential computing may add cost and operational risk without addressing a meaningful attacker.

It also does not replace data minimization, authorization, safe outputs, model evaluation, or incident response.

## Define the asset and threat model

Inventory each artifact:

| Artifact | Confidentiality concern | Typical control |
| --- | --- | --- |
| Raw document or image | Customer/PHI exposure | Minimize, regional encrypted store, short retention |
| Prompt/context | Reveals task and source data | Regional processing, no raw default logs |
| Embedding | May reveal source membership/content | Tenant scope, regional index, deletion lineage |
| Model weights | Vendor or company IP | Encrypted artifacts, measured loader, access control |
| Adapter/fine-tune | May memorize tenant data | Tenant isolation, evaluation, lifecycle policy |
| Output | Can reproduce sensitive input | Authorization, redaction, purpose controls |
| KV cache | Contains derived prompt state | Memory isolation, zeroization, tenant-safe reuse |
| Telemetry | IDs, lengths, timing, snippets | Aggregation, redaction, regional retention |
| Snapshot/crash dump | Plaintext memory risk | Disable or encrypt under separate policy |
| Key material | Unlocks every protected artifact | KMS/HSM, attested release, separation of duties |

Name attackers and exclusions explicitly.

A common confidential-inference threat model protects workload memory from the host operating system, hypervisor, and infrastructure administrator. It may not protect against:

- malicious application code inside the trusted environment;
- a vulnerable or backdoored model server;
- compromised build pipeline;
- side channels outside the platform guarantee;
- denial of service;
- output exfiltration through an authorized request;
- compromised client;
- incorrect model behavior;
- compelled access by an entity that controls keys or deployment.

Security claims must match the actual hardware, firmware, driver, accelerator, and cloud configuration.

## Confidential-computing building blocks

### Trusted execution environment

A TEE creates an isolated execution boundary whose memory is protected by hardware mechanisms. Depending on the technology, the boundary may cover a process, virtual machine, CPU memory, accelerator memory, or some combination.

Document:

- protected components;
- excluded components;
- device/driver and DMA path;
- memory encryption and integrity properties;
- debug and snapshot settings;
- supported attestation evidence;
- firmware and microcode trust;
- known side-channel assumptions;
- reset and zeroization behavior.

“Confidential VM” does not automatically mean the GPU, host networking, storage path, or observability agent is inside the same trust boundary.

### Measured launch

A measured launch produces cryptographic measurements of selected firmware, boot components, configuration, and workload artifacts.

Measurements identify what launched. They do not prove:

- the code is vulnerability-free;
- the model is accurate;
- policy is correct;
- no sensitive output will be generated;
- the measurement allowlist was approved wisely.

### Remote attestation

Attestation lets a verifier evaluate signed evidence about the platform and workload.

A safe flow:

1. client or workload obtains fresh attestation evidence;
2. verifier checks signature chain, nonce/freshness, platform state, measurements, security version, and debug policy;
3. verifier evaluates those claims against a versioned allowlist;
4. policy service issues a short-lived authorization;
5. KMS/HSM releases a scoped data key only to the approved workload identity and evidence;
6. workload decrypts the minimum required data in protected memory;
7. key use and policy revision are recorded;
8. keys and plaintext state are zeroized at request or instance teardown.

Bind evidence to a nonce or secure session so an old attestation cannot be replayed.

Attestation is an input to authorization, not authorization by itself.

### Attested key release

Keep data encryption keys outside the workload until the platform proves an approved state. Release:

- a short-lived key;
- for one tenant, purpose, dataset, region, workload measurement, and time window;
- to a workload identity bound to attestation;
- with a revocation and expiry mechanism.

Avoid releasing a broad master key to every instance. Envelope encryption keeps per-object or per-tenant keys scoped.

If the attestation service is unavailable, decide whether to use an already valid local lease, queue work, or fail closed. Do not silently bypass verification.

## CPU, accelerator, and I/O boundary

AI inference frequently crosses CPU and GPU/accelerator memory. The complete plaintext path matters:

1. encrypted object reaches regional storage;
2. approved workload obtains key;
3. CPU decrypts or streams data;
4. tokenizer/preprocessor creates tensors;
5. tensors move through drivers and interconnect;
6. accelerator stores inputs, weights, KV cache, and outputs;
7. output returns to CPU;
8. response filtering and serialization run;
9. memory is reclaimed or zeroized.

Ask at each step:

- Is plaintext protected?
- Which privileged software can read it?
- Is the buffer shared?
- Can it be paged, snapshotted, dumped, or inspected?
- Is DMA protected?
- Are accelerator resets and memory reuse safe?
- Does telemetry capture content?

If only CPU memory is protected but tensors are exposed to a host-controlled accelerator path, the end-to-end confidential-inference claim is incomplete.

## Residency as a data-flow property

Residency is not a region dropdown. Map the entire lifecycle:

- upload endpoint;
- web application and API gateway;
- message queue;
- object/database/vector/time-series storage;
- inference and safety services;
- cache and KV cache;
- telemetry and support tools;
- moderation or evaluation service;
- backups and replicas;
- disaster-recovery site;
- model-training or feedback pipeline;
- third-party subprocessor;
- administrator workstation and exported reports;
- deletion and legal-hold process.

Classify both payload and metadata. Document names, identifiers, embeddings, prompt hashes, IP addresses, and trace attributes may be restricted even if raw prompts are not stored.

Route using trusted tenant and data-class metadata before reading content. Content classification can refine policy, but it cannot be the only residency control.

## Sovereignty beyond residency

A sovereign deployment may require some combination of:

- local legal entity and contractual control;
- in-country or customer-controlled key custody;
- local operations and support personnel;
- restricted administrator nationality or clearance;
- isolated management plane;
- local identity provider and certificate authority;
- local software/artifact registry;
- local monitoring, ticketing, and backup;
- controlled update approval;
- ability to operate during disconnection from global services;
- customer approval for remote support;
- auditable subcontractor chain;
- exit and portability plan.

These requirements are customer- and jurisdiction-specific. Do not claim “sovereign” without a written control matrix and evidence.

A deployment operated locally but dependent on a foreign global control plane for every key, login, or policy decision may not satisfy operational sovereignty during disconnection.

## Deployment patterns

### Approved hosted model endpoint

Use when the provider contract, region, retention, data-use terms, identity, networking, and audit satisfy policy.

Benefits: fastest access to managed models and scaling.

Risks: provider control, limited attestation, opaque software changes, external logging, and restricted debugging evidence.

### Private managed deployment

A model is deployed in a tenant or provider-isolated account/network with private connectivity and scoped keys.

Benefits: stronger network/account isolation and control.

Risks: managed service operators or control planes may remain in the trust boundary.

### Confidential VM or confidential container

Run inference inside an attested CPU TEE.

Benefits: protects CPU memory from selected infrastructure threats.

Risks: performance, image/driver compatibility, attestation operations, and incomplete accelerator coverage.

### Confidential accelerator deployment

CPU and accelerator paths provide supported confidential-computing guarantees.

Benefits: protects more of the inference path.

Risks: hardware availability, driver/firmware complexity, feature limits, attestation chain, performance, and immature operations.

### Customer or sovereign cloud

Run in customer-controlled or jurisdiction-specific infrastructure with local keys and operations.

Benefits: control and residency.

Risks: capacity constraints, upgrade burden, model-serving expertise, slower access to new models, and divergence from the main platform.

### On-premises or disconnected deployment

Use when external connectivity or provider control is unacceptable.

Benefits: strongest local operational control.

Risks: GPU procurement, patching, supply chain, physical security, capacity, disaster recovery, and model lifecycle.

Choose the smallest pattern that satisfies the documented threat and sovereignty requirements.

## Reference architecture

A confidential regional inference cell contains:

- trusted ingress with workload identity;
- regional policy enforcement;
- encrypted regional source stores;
- attestation verifier;
- regional or customer-controlled KMS/HSM;
- approved artifact registry;
- attested model-serving pool;
- private networking and restricted egress;
- regional audit and telemetry;
- regional queue for unavailable capacity;
- local break-glass process;
- approved recovery path.

Request flow:

1. application authenticates and supplies tenant, purpose, data class, and required region;
2. gateway validates authorization and selects an eligible confidential route;
3. scheduler places work only on a compatible attested pool;
4. verifier challenges the workload and checks fresh evidence;
5. KMS releases a scoped key under attestation and workload policy;
6. encrypted context is fetched through private regional paths;
7. preprocessing, inference, and postprocessing occur inside the documented boundary;
8. output policy minimizes and authorizes the response;
9. plaintext buffers and request keys are destroyed;
10. regional evidence records measurements, policy, model, key grant, and lifecycle outcome;
11. only approved aggregates leave the region.

The gateway lesson governs route selection; this lesson governs whether the selected execution environment deserves the required trust claim.

## Workload identity and authorization

Use workload identity rather than node IP or static secrets. Bind:

- service identity;
- tenant;
- environment;
- region;
- workload measurement;
- deployment revision;
- purpose;
- requested key scope;
- attestation result.

The policy decision should be reproducible from versioned inputs.

Separate duties:

- platform team publishes infrastructure;
- model team publishes signed model artifacts;
- security team approves measurements and security versions;
- data owner approves datasets/purposes;
- key custodian defines release policy;
- operators cannot both change code and broaden key access unnoticed.

Emergency access must be time-bounded, approved, recorded, and tested—not a permanent bypass.

## Artifact and supply-chain trust

Attestation is only useful when measurements map to trusted artifacts.

Protect:

- source and build provenance;
- container/base image;
- model weights;
- tokenizer and preprocessing code;
- drivers, firmware, and runtime;
- policy bundle;
- adapter and postprocessor;
- evaluation evidence.

Use immutable digests and signed manifests. Scan dependencies, but do not equate a clean scan with trust.

Promote a release bundle only after security tests, workload evaluations, compatibility tests, and attestation-policy update. Rollback requires compatible measurements and models, not just an older tag.

If a vulnerability affects a trusted measurement:

1. stop new key grants to vulnerable launches when risk requires;
2. drain or isolate instances;
3. preserve minimal evidence;
4. publish patched artifacts and new measurements;
5. re-attest;
6. rotate affected keys if exposure is plausible;
7. identify requests processed by affected versions;
8. notify and remediate under policy.

## Key architecture

Use hierarchical envelope encryption:

- root keys in HSM/KMS;
- separate keys by environment and region;
- tenant or data-domain key-encryption keys;
- short-lived data-encryption keys for objects or sessions;
- independent audit and rotation.

Customer-managed keys can improve control, but “bring your own key” does not mean the customer exclusively controls plaintext. Clarify:

- who can invoke decrypt;
- which services are authorized;
- whether provider administrators can change the calling workload;
- whether keys leave the customer boundary;
- how revocation behaves;
- what remains in caches, logs, backups, or active memory.

Hold-your-own-key patterns may add stronger external custody, but availability and recovery become customer dependencies.

Key revocation prevents future decryption; it does not erase plaintext already exported or model outputs already stored.

## Network and egress controls

Prefer private connectivity and default-deny egress.

Allow only:

- approved regional data stores;
- attestation and key services;
- artifact repository;
- required identity endpoints;
- privacy-safe telemetry destinations;
- explicitly approved provider APIs.

Protect DNS and service discovery. Use workload-level policy, not only a broad subnet rule.

A TEE does not stop malicious code inside the TEE from exfiltrating through an allowed network. Restrict destinations, inspect metadata where permitted, cap response size, and monitor unusual transfer patterns.

For disconnected operation, mirror identities, artifacts, policies, revocation state, time sources, and observability components locally. Define how trust expires when synchronization is unavailable.

## Logging, observability, and debugging

Confidential systems are harder to debug because memory inspection and crash dumps undermine the guarantee.

Use:

- structured events without raw inputs;
- request and artifact digests;
- approved counters and latency histograms;
- attestation result and measurement IDs;
- key-grant decision and policy revision;
- model/runtime version;
- bounded sampled redacted content only under explicit purpose;
- regional trace storage;
- privacy-safe export aggregation.

Disable or protect:

- core dumps;
- swap;
- debug consoles;
- host-level profilers that expose memory;
- verbose tokenizer/model logs;
- support bundles containing prompts;
- unencrypted snapshots.

Create a synthetic-data reproduction path. Break-glass debugging of production plaintext requires explicit authorization, narrow scope, dual control, expiry, and audit; some threat models should prohibit it entirely.

Monitor security and availability separately. A healthy endpoint with failed attestation is not eligible.

## Availability and disaster recovery

Confidential controls introduce new dependencies:

- attestation verifier;
- endorsement/certificate service;
- KMS/HSM;
- approved measurements;
- secure time;
- specialized hardware capacity.

Define behavior for each outage.

Options:

- use a short-lived cached attestation/key lease for already running healthy instances;
- queue asynchronous work;
- route to another preapproved cell;
- serve a reduced non-sensitive capability;
- fail closed.

Never fall back from confidential processing to ordinary processing silently.

Replicate encrypted data only to approved locations. Recovery keys, backups, metadata, and operational access must meet the same sovereignty policy.

Test restore into an attested environment. A backup that can only be restored through an unauthorized global service does not satisfy the design.

## Multi-tenant isolation

A TEE can still contain several tenants, but shared memory, KV caches, batching, and model-server bugs expand the blast radius.

Isolation choices:

- shared attested pool with strict application isolation;
- separate process/container;
- separate confidential VM;
- separate accelerator partition;
- dedicated tenant cell.

Choose by data sensitivity, customer contract, model-serving capability, side-channel assumptions, scale, and cost.

Batch only requests whose policy permits co-residency. Never reuse KV cache across tenants unless the cache design proves exact scope and privacy.

Partition logs, keys, queues, caches, and usage records. Tenant deletion must propagate to regional artifacts and derived data.

## Privacy-preserving techniques

Confidential computing protects execution infrastructure; other techniques address different risks:

- **data minimization:** remove unnecessary fields before inference;
- **tokenization/pseudonymization:** replace direct identifiers where workflow permits;
- **differential privacy:** bound information leakage from aggregate statistics or training;
- **federated learning:** keep raw training data local while sharing updates, with additional privacy/security controls;
- **secure multi-party computation:** compute jointly without revealing inputs, often with high cost;
- **homomorphic encryption:** compute on encrypted values for limited workloads, currently impractical for many full-scale generative-inference paths;
- **split processing:** perform sensitive preprocessing locally and send only the necessary representation, after evaluating reconstruction risk.

Do not market one technique as solving every privacy problem. Embeddings and gradients can still leak information.

## Output and inference risks

Protecting the runtime does not make the model safe.

The model may:

- reveal data from the current context to an unauthorized user;
- reproduce training data;
- follow prompt injection;
- infer sensitive attributes;
- encode data in tool arguments or outputs;
- return excessive source text;
- create insecure actions.

Authorization must happen before retrieval, context assembly, and output release. Apply purpose limitation, citation controls, redaction, output size limits, tool approval, and evaluation.

A confidential enclave faithfully running malicious prompt content is still unsafe.

## Construction operations example

A public infrastructure owner permits contract documents, labor records, security drawings, and bid information to be processed only in an approved US region. The owner also requires customer-controlled keys and no infrastructure administrator access to plaintext.

Design:

1. classify documents by project, purpose, and sensitivity at ingestion;
2. encrypt each object with a project-scoped data key under an owner-controlled regional key;
3. keep document, vector, queue, audit, and backup data in approved regions;
4. route sensitive inference only to an attested regional pool with approved CPU/accelerator coverage;
5. verify fresh attestation and release short-lived project/purpose keys;
6. load signed model, tokenizer, adapter, and policy artifacts by digest;
7. preprocess drawings and contracts inside the protected boundary;
8. disable raw prompt logs, dumps, and unapproved support access;
9. export only authorized answers, citations, and privacy-safe metrics;
10. return ERP or project-system proposals to a durable approval workflow;
11. revalidate source versions and authorization before any action;
12. zeroize request state and record key/attestation evidence;
13. queue work if attestation, key custody, or approved capacity is unavailable.

A subcontractor user authorized for one specification section must not gain access to security drawings merely because both are decrypted inside the same enclave. Application authorization and tenant/project boundaries remain mandatory.

## Healthcare operations variation

A healthcare operations assistant extracts prior-authorization packet fields and summarizes operational records containing PHI.

The organization requires:

- processing in approved US regions;
- PHI excluded from global telemetry;
- customer-managed regional keys;
- attested processing for sensitive workloads;
- no training on prompts;
- bounded retention;
- approved disaster recovery;
- audited support access.

The system minimizes the record before inference, retrieves only authorized encounter documents, releases keys only to an approved measured workload, and stores regional evidence without raw PHI. Outputs are sent only to the authorized workflow and are not clinical decisions.

If the confidential accelerator pool is full, the request waits or uses a separately approved confidential cell. It does not fall back to a standard endpoint.

If an emergency support investigation needs content, policy determines whether dual-controlled, time-limited access is allowed. The exception, evidence accessed, and outcome are audited.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Hosted approved endpoint | Fastest managed capability | Provider/control-plane trust |
| Confidential VM | Host-memory protection | Compatibility and overhead |
| Confidential accelerator | Broader inference protection | Hardware scarcity and complexity |
| Customer/sovereign cloud | Local operational control | Upgrade and capacity burden |
| On-premises | Maximum local control | High lifecycle responsibility |
| Shared attested pool | Utilization and batching | Larger tenant blast radius |
| Dedicated tenant cell | Stronger isolation | Cost and fragmentation |
| Attested key release | Keys follow measured state | Verifier/KMS availability |
| Cached short key lease | Survives brief outage | Revocation delay |
| Customer-managed key | Better custody and revocation | Integration/recovery complexity |
| Hold-your-own-key | Strong external control | Availability dependency |
| Raw debug access | Faster incident diagnosis | Breaks confidentiality claim |
| Synthetic diagnostics | Preserves privacy | Harder reproduction |
| Strict local operations | Strong sovereignty | Staffing and feature lag |
| Global managed control plane | Operational efficiency | Jurisdiction/disconnection risk |
| No sensitive fallback | Preserves policy | Lower availability |
| Cross-region approved cell | Resilience | Duplicate operations and validation |
| Homomorphic/MPC techniques | Strong cryptographic privacy | Performance and feature limits |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Region selected, global logs enabled | Residency violation | End-to-end flow inventory |
| Encryption marketed as data-in-use protection | Host can read plaintext | Explicit TEE threat model |
| CPU protected, accelerator path exposed | Incomplete confidential claim | Verify end-to-end boundary |
| Attestation accepted without nonce | Replay of old evidence | Fresh challenge/session binding |
| Measurement allowlist too broad | Unapproved code receives keys | Exact policy and review |
| Attestation treated as code quality | Vulnerable code trusted | Supply-chain and security testing |
| Debug mode permitted | Memory confidentiality weakened | Enforce debug-off claim |
| Broad master key released | Large blast radius | Scoped short-lived keys |
| Static key in image | Artifact compromise | External KMS/HSM release |
| Key release ignores tenant/purpose | Cross-scope decryption | Bind full authorization context |
| TEE code has open egress | Plaintext exfiltration | Default-deny destinations |
| Raw prompts in crash dump | Sensitive data exposed | Disable/protect dumps |
| Global support copies artifacts | Sovereignty breach | Local support workflow |
| Backup stored correctly but restored elsewhere | Recovery policy breach | Attested regional restore tests |
| Failover uses ordinary endpoint | Silent loss of protection | Hard route eligibility |
| Stale trusted measurement after vulnerability | Known weakness remains authorized | Revoke and patch |
| Firmware endorsement unavailable | New nodes cannot attest | Capacity reserve and outage policy |
| Shared KV cache crosses tenants | Context leakage | Tenant-scoped cache or disable |
| Model output bypasses authorization | Confidential input leaked legitimately | Output policy and user authorization |
| Deletion stops at source | Embedding/cache remains | Lineage-driven regional deletion |
| Sovereign label lacks control matrix | False assurance | Written claims and evidence |
| Customer key assumed exclusive control | Provider can invoke decrypt | Analyze caller and admin control |
| Global time/identity dependency fails | Sovereign cell cannot operate | Local bounded dependencies |
| Side-channel risk omitted | Overstated guarantee | Document platform assumptions |

## What to measure

Track by region, tenant, workload, release, and trust profile:

- percentage of sensitive requests routed to an eligible confidential cell;
- attestation success, rejection reason, freshness, and verification latency;
- key-grant success, scope, lease age, and denied grants;
- approved-measurement coverage and vulnerable-measurement revocation time;
- confidential CPU/accelerator capacity, queue time, and fallback denial;
- percentage of plaintext lifecycle components covered by the threat model;
- regional data-flow and backup/restore compliance;
- unapproved egress attempts;
- debug, dump, and snapshot policy violations;
- raw-sensitive-data findings in logs and support bundles;
- tenant co-residency and cache-isolation tests;
- model/artifact signature and provenance verification;
- deletion propagation across embeddings, caches, telemetry, and backups;
- emergency-access frequency, scope, expiry, and review;
- regional recovery time and restore evidence;
- confidential-versus-standard performance and cost;
- output authorization, citation leakage, and tool-safety outcomes.

A strong metric is not “TEE enabled.” It is “every sensitive request and artifact followed an evidenced, authorized path consistent with the declared threat model.”

## Interview prompt

> Design a confidential, sovereign AI inference platform for construction and healthcare operations. Sensitive documents and PHI must remain in approved regions, infrastructure administrators must not see plaintext, customers control keys, accelerators serve large models, and the service must survive failures without silently weakening protection.

Cover the threat model, data lifecycle, CPU/accelerator boundary, attestation, key release, identity, artifact trust, network egress, multi-tenancy, observability, debugging, residency, sovereignty, disaster recovery, output controls, and operational tradeoffs.

## Worked answer

**One-line conclusion:** Treat confidential inference as an attested, least-privilege data lifecycle—not a hardware checkbox—binding region, workload measurement, identity, purpose, key scope, network path, and output authorization before plaintext is released.

“I would begin with a control matrix. For every prompt, document, embedding, model artifact, output, log, and backup, I name its allowed regions, administrators, key custodian, retention, and recovery path. I define the attacker: the host operator and infrastructure administrator are outside the plaintext trust boundary, while approved application code remains trusted and must be secured separately.

Sensitive requests route only to a regional confidential cell. The cell includes trusted ingress, policy, attestation verification, customer-controlled regional KMS, signed artifacts, protected CPU and accelerator paths, default-deny egress, and regional telemetry. I document exactly which memory, drivers, interconnects, and devices the hardware guarantee covers.

At measured launch, the platform produces fresh evidence. The verifier checks the signature chain, nonce, security version, debug state, and exact workload measurements against an approved release. The KMS then grants only a short-lived tenant, dataset, purpose, and region-scoped key to that workload identity. No valid attestation means no new key.

Artifacts are immutable and signed: model, tokenizer, runtime, driver, adapter, and policy. A vulnerable measurement is revoked, drained, patched, re-attested, and traced to affected requests. Plaintext logs, dumps, swap, and unapproved profiling are disabled. Structured regional evidence records measurements, key policy, versions, and outcomes without raw data.

Multi-tenancy is explicit. High-risk tenants receive dedicated cells; shared pools use tenant-scoped queues, keys, caches, and memory zeroization. Application authorization still controls retrieval and output because a TEE does not prevent an authorized model process from leaking data.

If capacity or attestation fails, I queue, use another preapproved confidential region, or fail closed. I never fall back to ordinary inference silently. Backups and restore exercises obey the same location, key, and attestation policy. Healthcare PHI stays out of global telemetry, and construction actions remain in approval-bound workflows outside the model.

I measure end-to-end protected-path coverage, attestation/key denials, stale measurements, capacity, unapproved egress, sensitive logging, deletion, recovery, and output leakage—not simply whether confidential computing is enabled.”

## Practice lab

Design the platform when:

- contract documents and PHI require approved US processing;
- infrastructure administrators are outside the plaintext trust boundary;
- the CPU supports a TEE, but one GPU path has no confidential mode;
- the attestation service is unreachable for 20 minutes;
- an already running instance has a key lease with 8 minutes remaining;
- a firmware vulnerability invalidates the current measurement;
- the customer controls the regional root key;
- a support engineer requests a memory dump;
- the model server needs an external tokenizer package;
- global telemetry includes request IDs and document titles;
- the primary confidential GPU pool is full;
- a disaster-recovery region has encrypted backups but an unapproved operations team;
- two tenants share batching and KV-cache infrastructure;
- a user is authorized for one project but asks about another;
- an output quotes an entire sensitive contract section;
- a source-deletion request arrives after embeddings and evaluation samples were created.

Specify assets, attackers, trust boundary, attestation claims, key-release policy, CPU/GPU path, workload identity, signed artifacts, egress, logs, debugging, isolation, regional routing, outage behavior, DR, deletion, output authorization, and evidence.

Then explain which guarantees you cannot make and how you would communicate them to a customer without using “confidential” or “sovereign” as an undefined label.

## References

- [IETF RFC 9334 — Remote ATtestation procedureS Architecture](https://www.rfc-editor.org/rfc/rfc9334)
- [Confidential Computing Consortium](https://confidentialcomputing.io/)
- [NIST SP 800-207 — Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [NIST SP 800-57 Part 1 Revision 5 — Key Management](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [AMD SEV-SNP documentation](https://www.amd.com/en/developer/sev.html)
- [Intel Trust Domain Extensions](https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html)
- [NVIDIA Confidential Computing](https://www.nvidia.com/en-us/data-center/solutions/confidential-computing/)
