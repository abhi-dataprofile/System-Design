# Secure code execution, sandboxing, and untrusted computation

AI agents can generate Python, SQL, shell commands, spreadsheet formulas, data transformations, and document-processing code. Executing that code can make an assistant much more useful—and can turn a prompt injection into remote code execution, data theft, infrastructure compromise, or denial of service.

Generated code is untrusted even when the model is trusted. Uploaded code, third-party packages, document parsers, notebooks, macros, and tool outputs are untrusted too.

A production sandbox is not “run the command in Docker.” It is a complete execution system with an explicit threat model, small capabilities, strong isolation, bounded resources, controlled data staging, reproducible environments, validated outputs, and disposable state.

## Mental model

Treat code execution as a capability lease:

[
Execution = Identity + Image + Inputs + Capabilities + Limits + Time + Evidence
]

Each job receives only:

- an authenticated workload identity;
- an immutable runtime image;
- explicit input artifacts;
- a read/write filesystem map;
- approved network destinations;
- no ambient secrets;
- CPU, memory, storage, process, GPU, and time budgets;
- a finite lifetime;
- a declared output contract;
- a complete audit record.

The sandbox should assume the code will try to:

- read files it was not given;
- inspect environment variables and metadata services;
- escape the runtime;
- create many processes;
- consume all memory, disk, CPU, GPU, or network;
- install malicious dependencies;
- exfiltrate data through allowed channels;
- poison outputs and caches;
- persist after the job;
- attack neighboring tenants.

Security comes from removing ambient authority and layering isolation—not from asking the model whether its code is safe.

## When to use it

Use a sandbox when the system executes:

- model-generated Python, JavaScript, shell, SQL, or notebook cells;
- user-uploaded scripts;
- document converters and media tools over untrusted files;
- spreadsheet formulas or macros;
- compiler/build/test workloads;
- browser automation with downloaded content;
- plugins or third-party extensions;
- data-analysis jobs over tenant records;
- dynamic package installation;
- evaluation code submitted by users or models.

Avoid arbitrary code when a fixed typed tool can solve the task. A “sum these invoice amounts” tool is safer, cheaper, and more observable than a general Python interpreter.

Use the least expressive mechanism:

1. deterministic library function;
2. declarative query/template;
3. restricted domain-specific language;
4. WebAssembly or constrained runtime;
5. isolated process/container;
6. microVM or dedicated VM;
7. dedicated tenant hardware for the highest-risk cases.

More expressiveness requires a stronger boundary.

## Threat model

Define what the sandbox protects.

### Assets

- host kernel and control plane;
- other tenants and jobs;
- source documents and databases;
- credentials, tokens, and metadata services;
- internal networks and external systems;
- model weights and proprietary code;
- output stores and caches;
- GPU memory;
- billing and compute capacity;
- audit evidence.

### Attackers

- malicious user;
- malicious document author using indirect prompt injection;
- compromised package;
- accidental model-generated bug;
- malicious insider submitting a job;
- parser exploit;
- colluding tenants;
- compromised base image or registry.

### Boundaries

Document whether isolation relies on:

- language runtime;
- process restrictions;
- Linux namespaces/cgroups/seccomp;
- user-space kernel;
- WebAssembly runtime;
- microVM/hypervisor;
- dedicated VM/node;
- hardware confidential computing.

Containers share a host kernel. They are a packaging and isolation mechanism, not the same boundary as a VM. Rootless execution and hardened profiles reduce risk but do not remove kernel exposure.

## Architecture

A safe execution service separates control and execution planes.

### Control plane

The control plane:

- authenticates caller and delegated subject;
- authorizes tenant, purpose, runtime, and data;
- validates job manifest;
- selects isolation class;
- resolves signed runtime image;
- stages immutable inputs;
- assigns limits and network policy;
- creates a unique job identity;
- schedules onto an appropriate worker pool;
- records policy and artifact versions;
- collects status and evidence;
- destroys the environment.

It does not run untrusted code in its own process.

### Execution plane

The execution plane contains disposable workers:

- isolated per job or trusted batch;
- no control-plane credentials;
- default-deny network;
- read-only base image;
- ephemeral writable filesystem;
- non-root user;
- reduced kernel capabilities;
- syscall filter;
- process/resource limits;
- explicit input/output mounts;
- monitored lifecycle;
- forced teardown and zeroization.

Separate pools by isolation/risk class. Do not place low-trust public code beside sensitive PHI or financial workloads merely because both fit on the same node.

## Job manifest

A versioned job request should include:

- tenant and delegated subject;
- workflow/purpose;
- code or code artifact digest;
- runtime image digest;
- entry point;
- language/runtime version;
- input artifact IDs and versions;
- output schema and size limit;
- allowed filesystem paths;
- allowed network destinations/methods;
- environment variables by allowlist;
- secret handles, normally none;
- CPU, memory, process, thread, disk, I/O, GPU, and wall-clock limits;
- region/data classification;
- isolation profile;
- package/dependency policy;
- idempotency/job ID;
- retention and deletion policy.

The model may propose code and requested libraries. Trusted code constructs and authorizes the final manifest.

## Isolation layers

### Language-level restrictions

Examples include a restricted expression evaluator or safe query subset.

Benefits:

- fast;
- low overhead;
- narrow semantics.

Risks:

- language escape vulnerabilities;
- dangerous reflection/import features;
- denial through pathological expressions;
- parser inconsistencies.

Do not create a “safe Python” by deleting a few built-ins. General-purpose languages have large escape surfaces.

### WebAssembly

WebAssembly with a capability-oriented system interface can provide a small, portable runtime boundary. The module receives only preopened directories, clocks, random sources, sockets, or other granted capabilities.

Benefits:

- fast startup;
- compact distribution;
- explicit host functions;
- cross-platform.

Risks:

- runtime vulnerabilities;
- incomplete language/library support;
- host-function mistakes;
- resource metering complexity;
- native extensions unavailable.

### Hardened container

Use namespaces, cgroups, seccomp, capabilities, read-only filesystems, non-root users, and mandatory access control.

Benefits:

- mature ecosystem;
- broad language compatibility;
- fast startup.

Risks:

- shared host kernel;
- container runtime and kernel escape;
- misconfigured mounts, devices, or capabilities;
- node-level side channels.

### User-space kernel

A user-space kernel intercepts or implements system calls between the workload and host kernel, reducing direct kernel attack surface.

Benefits:

- stronger container isolation without a full guest OS.

Risks:

- compatibility gaps;
- performance overhead;
- another complex security boundary.

### MicroVM or VM

A hardware-virtualized boundary gives each job or small group a separate guest kernel.

Benefits:

- stronger isolation from the host and neighboring jobs;
- broad runtime support.

Risks:

- startup and memory overhead;
- guest image lifecycle;
- hypervisor/device attack surface;
- slower scaling.

### Dedicated node or confidential VM

Use for very high-value or regulated workloads where co-tenancy or infrastructure trust is unacceptable.

Benefits:

- smaller co-tenant blast radius;
- optional data-in-use protection.

Risks:

- high cost;
- capacity fragmentation;
- operational complexity.

Choose based on attacker capability, data sensitivity, code origin, and consequence.

## Filesystem design

Use a read-only immutable root image.

Mount only:

- a read-only input directory containing authorized artifacts;
- a writable ephemeral scratch directory with quota;
- a separate output directory with size and file-count limits.

Do not mount:

- host root;
- container runtime socket;
- orchestration credentials;
- shared home directory;
- cloud credentials;
- package-manager credentials;
- unrelated tenant volumes;
- sensitive device nodes.

Prevent path traversal and symlink attacks when staging or collecting files. Resolve paths against trusted roots and reject special files, hard links, sockets, and unexpected ownership.

Archive extraction needs limits on:

- compressed and expanded bytes;
- file count;
- nesting depth;
- path length;
- compression ratio;
- sparse files;
- links;
- duplicate names;
- character normalization.

A ZIP is code-adjacent input because parsers and extraction logic are attack surfaces.

## Network and egress

Default to no network.

If a job needs access, issue explicit capabilities:

- destination host/service identity;
- port/protocol;
- method;
- request count;
- byte limit;
- time window;
- DNS behavior;
- data classification;
- response size.

Route through an egress proxy that authenticates the job and enforces policy. Do not allow arbitrary IPs, redirects, URL schemes, or DNS rebinding.

Block:

- cloud metadata services;
- control-plane endpoints;
- private network ranges unless explicitly required;
- localhost services outside the sandbox;
- raw sockets;
- peer-to-peer job traffic;
- unapproved package registries;
- data-exfiltration destinations.

An allowlisted domain is not enough if redirects, DNS, uploads, or attacker-controlled subdomains are possible.

For database analysis, prefer a brokered query API with row/column/tenant policy over direct database credentials.

## Secrets and credentials

The safest sandbox has no secrets.

If access is necessary:

1. authorize job, subject, purpose, resource, and operation;
2. issue a short-lived audience-specific credential;
3. bind it to the sandbox workload identity where possible;
4. expose it through a broker, not a general environment variable;
5. prevent logs, process listings, and crash dumps from capturing it;
6. revoke at job end;
7. audit use.

Prefer a tool or data proxy so untrusted code never receives the credential itself.

Never pass provider keys, ERP credentials, user tokens, or cloud credentials into generated code.

## Dependency and package handling

Dynamic installation is a supply-chain and network risk.

Safer patterns:

- prebuild approved runtime images;
- use a curated internal registry;
- pin exact versions and hashes;
- verify signatures/provenance where available;
- scan and review packages;
- disable lifecycle scripts when feasible;
- build wheels/artifacts in a separate sandbox;
- cache artifacts by digest;
- prohibit native extensions for lower-trust tiers;
- record dependency lockfile and resolver version.

Package names can be hallucinated or dependency-confused. A model-generated “pip install company-helper” must not reach the public registry automatically.

Do not allow package installation during high-privilege jobs. Promote dependencies through a controlled image pipeline.

## Resource controls

Set hard limits on:

- wall-clock and CPU time;
- memory and swap;
- process/thread count;
- file descriptors;
- filesystem bytes and inodes;
- read/write IOPS;
- network requests and bytes;
- stdout/stderr size;
- output files and bytes;
- GPU time/memory;
- system calls;
- recursion and child jobs.

Use both per-job and per-tenant quotas. A job killed for memory must not destabilize the node.

Bound compilation and parsing before inference. Reject oversized code, deeply nested ASTs, catastrophic regular expressions, and generated output explosions.

Timeout must terminate descendants, not just the parent process. Use a supervisor and verify cleanup.

## Scheduling and multi-tenancy

Classify workloads:

- trusted platform code;
- tenant-authored code;
- model-generated code;
- anonymous/public code;
- sensitive-data code;
- GPU/native-extension code.

Schedule high-risk classes onto stronger isolation pools.

Prevent:

- noisy-neighbor starvation;
- CPU/GPU overcommit beyond policy;
- cross-tenant shared writable caches;
- reuse of dirty memory/state;
- jobs on stale or vulnerable images;
- sensitive and public jobs sharing unsafe devices.

Warm pools improve latency but risk residual state. Reset the full isolation boundary, rotate job identity, clear disks and memory where supported, and test for cross-job artifacts.

## Input staging and data minimization

The control plane should stage only authorized inputs.

For each artifact:

- verify tenant and resource authorization;
- bind version/digest;
- classify sensitivity;
- copy or mount read-only;
- rename to opaque safe paths;
- strip unnecessary metadata;
- set retention;
- record lineage.

Do not give a sandbox an entire project bucket when it needs two CSV files.

For PHI or confidential documents, keep the sandbox in an approved region and isolation class. Generated code cannot broaden the dataset by scanning storage.

## Output collection

Outputs are untrusted.

Before publication:

- enforce file type, count, and size;
- reject devices, links, executables, macros, and nested archives unless required;
- scan for malware and secrets;
- validate schema;
- verify encoding;
- remove active content;
- inspect provenance and job status;
- apply tenant and retention metadata;
- store in quarantine;
- require review for consequential use.

A CSV can contain spreadsheet formula injection. A notebook or HTML report can contain JavaScript. A PDF can contain actions or embedded files. “The sandbox produced it” does not make it safe.

Never execute generated output automatically in a less isolated environment.

## SQL and data analysis

Generated SQL should run through a policy layer:

- parse into an AST;
- allow only approved statement types;
- resolve tables/views within tenant scope;
- enforce row/column policy;
- set statement timeout and scan limit;
- use read-only transaction/replica;
- cap result rows/bytes;
- prohibit extensions, file access, network functions, and administrative commands;
- parameterize values;
- audit normalized query and outcome.

For writes, use typed business APIs and approval-bound workflows instead of arbitrary SQL.

A read-only database credential can still expose all data or cause expensive queries.

## Browser and document sandboxes

Browsers, PDF renderers, office converters, image libraries, and media tools process complex attacker-controlled formats.

Use:

- separate low-trust worker pools;
- current patched versions;
- no ambient network;
- file and page limits;
- disabled active content/macros;
- process isolation;
- strict time/memory;
- output quarantine;
- content digest and tool version;
- crash monitoring;
- rapid image revocation.

Do not run document conversion inside the main API or inference process.

## Reproducibility and evidence

Record:

- job and workflow IDs;
- actor and subject;
- policy/isolation profile;
- code digest;
- model/prompt/tool version that produced code;
- image and dependency digests;
- input IDs, versions, and digests;
- resource and network grants;
- start/end and termination reason;
- resource usage;
- syscalls/policy violations at an appropriate level;
- output digests and validation;
- region/node/runtime;
- retention and deletion linkage.

Avoid storing secrets or raw sensitive content in logs.

Reproducibility supports debugging and audit, but nondeterministic clocks, randomness, parallelism, external services, and floating dependencies must be captured or eliminated.

Provide synthetic-data replay rather than opening production sandboxes to interactive debugging.

## Side effects and two-phase design

Separate computation from action.

Phase 1 sandbox:

- read authorized inputs;
- calculate;
- produce a typed proposal;
- cannot call business write systems.

Phase 2 trusted workflow:

- validates output;
- shows evidence;
- obtains approval;
- binds exact payload;
- rechecks current state;
- executes through a narrow tool;
- applies idempotency;
- reconciles outcome.

Code execution should never directly approve invoices, send messages, update patient records, or control equipment.

## Failure handling

Plan for:

- sandbox startup failure;
- image pull/verification failure;
- dependency unavailable;
- timeout;
- out-of-memory kill;
- disk quota;
- output limit;
- policy violation;
- worker/node loss;
- runtime escape alert;
- incomplete cleanup;
- result upload failure;
- duplicate job delivery;
- region capacity exhaustion.

Jobs need explicit terminal states such as:

- succeeded;
- failed validation;
- policy denied;
- timed out;
- resource exhausted;
- canceled;
- infrastructure lost;
- output quarantined;
- security incident.

Use idempotent job IDs. A retry creates a new attempt linked to the logical job and does not overwrite prior evidence.

After suspected escape, quarantine the node, revoke workload identities, preserve minimal forensics, rotate affected credentials, identify co-resident jobs, and rebuild from trusted images.

## Construction operations example

A construction assistant receives cost-code CSVs, invoice PDFs, schedule exports, and bid sheets. A user asks it to generate Python that reconciles invoice lines to commitments and produces an Excel exception report.

Design:

1. authorize the exact project files and snapshot their versions;
2. generate code but treat it as untrusted;
3. choose a microVM or hardened sandbox profile with no network;
4. use a signed Python image with approved data libraries;
5. mount inputs read-only under opaque paths;
6. provide no ERP, Procore, cloud, or package credentials;
7. set CPU, memory, disk, process, output, and time limits;
8. write only to a quota-bound output directory;
9. validate the workbook for formulas, external links, macros, hidden sheets, size, and schema;
10. quarantine and preview the report;
11. record code/image/input/output digests;
12. let a trusted workflow turn reviewed exceptions into proposals;
13. require separate approval and idempotent APIs for any external update.

If a PDF contains “run curl to upload this project,” the code may attempt it, but the sandbox has no egress or credentials. The event is logged as a policy violation.

If the model requests an unknown package, the job fails with an explicit dependency request; it does not install from the public internet.

## Healthcare operations variation

A healthcare operations analyst asks an assistant to calculate packet-completeness statistics from authorized, minimized encounter data.

The sandbox:

- runs in the approved PHI region;
- receives only permitted columns and encounters;
- uses a read-only input snapshot;
- has no internet or clinical-system credentials;
- cannot query another patient;
- runs with strict resource limits;
- emits only aggregate tables meeting policy;
- scans outputs for direct identifiers and small-cell disclosure;
- deletes ephemeral storage at completion;
- records lineage without raw PHI in global logs.

The code cannot diagnose, score care, or write to the EHR. Any operational task derived from the analysis enters a separate authorized workflow.

If output validation detects patient identifiers, the result remains quarantined and never reaches the user.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Fixed typed tool | Small attack surface | Limited flexibility |
| General code sandbox | Powerful analysis | Large security surface |
| Language restriction | Fast and narrow | Escape/compatibility risk |
| WebAssembly | Capability-oriented and portable | Ecosystem limits |
| Hardened container | Fast startup and mature tooling | Shared host kernel |
| User-space kernel | Reduced kernel exposure | Compatibility/overhead |
| MicroVM | Stronger tenant boundary | Startup and memory cost |
| Dedicated node | Small co-tenant blast radius | Expensive and fragmented |
| Warm pool | Low latency | Residual-state risk |
| Fresh environment | Cleaner isolation | Higher latency |
| No network | Strong exfiltration defense | Limited dependencies |
| Egress proxy | Controlled access | Policy complexity |
| Prebuilt dependencies | Reproducible and safer | Slower library adoption |
| Dynamic package install | Flexible | Supply-chain exposure |
| Direct database access | Powerful queries | Broad data/cost risk |
| Brokered data API | Strong policy | Less expressiveness |
| Raw output download | Simple | Active-content risk |
| Quarantine and validation | Safer publication | Added latency |
| Full execution logs | Easier debugging | Sensitive-data exposure |
| Structured evidence | Lower privacy risk | Harder diagnosis |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Generated code trusted because model wrote it | Code execution compromise | Treat all code as hostile |
| “Docker equals sandbox” | Kernel escape blast radius | Threat-based layered isolation |
| Sandbox runs in API process | Full service compromise | Separate execution plane |
| Container runs privileged | Host control | No privilege/capabilities/devices |
| Runtime socket mounted | Container takeover | Never expose control sockets |
| Host path mounted read-write | Data/system damage | Explicit read-only mounts |
| Cloud metadata reachable | Credential theft | Network isolation |
| Broad environment secrets | Exfiltration | No ambient secrets |
| Public package install allowed | Dependency confusion/malware | Curated pinned registry |
| Floating image tag | Unreproducible/compromised run | Digest pinning/signatures |
| CPU limit without wall timeout | Sleeping job persists | Multiple hard limits |
| Parent killed, children remain | Resource leak/attack | Supervisor and cgroup teardown |
| Disk bytes limited, inodes unlimited | Node exhaustion | Bytes and inode quotas |
| Archive extracted blindly | Zip bomb/path traversal | Bounded safe extraction |
| DNS allowlist only | Rebinding/exfiltration | Egress proxy and identity |
| Read-only DB credential is broad | Data leak/expensive scan | Brokered policy and limits |
| Shared writable cache | Cross-tenant poisoning | Tenant/job scope |
| Warm worker not scrubbed | Residual data leak | Boundary reset/tests |
| GPU memory reused | Cross-job data exposure | Supported zeroization/isolation |
| Output trusted automatically | Formula/macro/script attack | Quarantine and validation |
| Sandbox can call write APIs | Direct business side effect | Compute/action separation |
| Logs capture input/secrets | Secondary breach | Structured redacted evidence |
| Retry overwrites prior evidence | Lost audit | Logical job plus attempts |
| Escape alert ignored | Co-resident compromise | Node quarantine/rotation |
| PHI job scheduled in wrong region | Policy violation | Hard regional eligibility |

## What to measure

Track by tenant, workload class, isolation profile, image, and region:

- jobs admitted, denied, queued, and completed;
- startup and execution latency;
- timeout, OOM, disk, process, and output-limit terminations;
- network and syscall policy violations;
- metadata-service access attempts;
- dependency requests and blocked packages;
- image signature/provenance failures;
- vulnerable/stale runtime usage;
- cross-job residue tests;
- tenant co-residency and isolation class;
- CPU, memory, storage, GPU, network, and cost per job;
- output quarantine and rejection reasons;
- formula, macro, executable, secret, and malware findings;
- jobs with reproducible code/image/input/output digests;
- cleanup and credential-revocation success;
- worker escape alerts and node quarantine time;
- data-deletion propagation;
- proposals produced versus external actions executed;
- security incidents by source: user, document, model, or dependency.

A successful service is not one where most code runs. It is one where hostile code cannot exceed its explicit capabilities and every accepted output remains untrusted until validated.

## Interview prompt

> Design a multi-tenant secure code-execution platform for an AI operations assistant. It runs model-generated Python and SQL over construction or healthcare data, processes untrusted documents, may request packages, and returns reports. It must prevent cross-tenant access, credential theft, network exfiltration, host escape, denial of service, and unsafe downstream actions.

Cover the threat model, isolation tiers, control and execution planes, workload identity, job manifest, filesystem, egress, secrets, dependencies, resource limits, scheduling, input staging, output quarantine, reproducibility, observability, failure recovery, and action separation.

## Worked answer

**One-line conclusion:** Run every generated or uploaded program as a disposable, capability-starved job with immutable inputs, signed runtime artifacts, no ambient credentials, default-deny egress, hard resource limits, validated outputs, and no direct business side effects.

“I would first ask whether a typed tool can replace arbitrary code. For the cases that need code, the application creates a trusted job manifest; the model only proposes code and dependencies.

The control plane authenticates actor and subject, authorizes exact tenant inputs, pins code and image digests, selects an isolation tier, assigns limits, stages read-only files, and creates a unique workload identity. It never executes untrusted code.

The execution plane uses disposable per-job microVMs or hardened containers according to risk. Jobs run non-root with a read-only base, ephemeral quota-bound scratch, reduced capabilities, syscall policy, process limits, no host mounts, and no orchestration or cloud credentials. Network is denied by default; exceptional access goes through an authenticated egress or data proxy with destination and byte limits.

Dependencies come from signed, curated, hash-pinned images. Unknown packages become review requests, not public-registry installs. Inputs are minimized and versioned. CPU, wall time, memory, processes, disk bytes/inodes, output, and network are all bounded, and teardown kills descendants.

Outputs enter quarantine. The platform validates type, schema, size, malware, secrets, formulas, macros, scripts, links, and tenant metadata before publication. Code can calculate a typed proposal but cannot call Procore, ERP, EHR, email, or building controls. A trusted approval-bound workflow handles actions separately.

I record job, actor, policy, isolation, code, image, dependency, input, output, resource, and termination evidence. Retries are new attempts. A suspected escape quarantines the node, revokes identities, examines co-resident jobs, and rebuilds from trusted artifacts.

For healthcare, the sandbox stays in the approved PHI region, receives minimum necessary data, has no internet, and checks outputs for identifiers. For construction, a malicious PDF instruction cannot exfiltrate project files because the job has neither network nor credentials.”

## Practice lab

Design the service when:

- an LLM generates Python over invoice CSVs and PDF text;
- one PDF instructs the code to upload all project files;
- the code requests a misspelled public package;
- two tenants have identically named input files;
- a ZIP expands from 50 MB to 2 TB;
- the script forks indefinitely;
- stdout grows without bound;
- a user requests direct database access;
- a warm worker contains a previous job’s temporary file;
- one workload needs a native extension and GPU;
- the cloud metadata service is reachable from the node;
- a generated workbook contains formulas and an external link;
- a job produces a valid report and an ERP-update script;
- the result upload succeeds but job status persistence fails;
- a runtime-escape alert fires on the shared node;
- a PHI analysis is accidentally routed to a non-approved region.

Specify isolation tiers, manifest, identity, mounts, egress, dependency policy, limits, scheduling, idempotency, output validation, evidence, cleanup, incident containment, and how computation remains separate from action.

Then explain which workloads can use WebAssembly, hardened containers, microVMs, or dedicated nodes—and why.

## References

- [NIST SP 800-190 — Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Linux seccomp documentation](https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html)
- [gVisor security model](https://gvisor.dev/docs/architecture_guide/security/)
- [Firecracker microVM documentation](https://firecracker-microvm.github.io/)
- [WebAssembly System Interface](https://wasi.dev/)
- [SLSA supply-chain framework](https://slsa.dev/)
- [OpenSSF Secure Supply Chain Consumption Framework](https://github.com/ossf/s2c2f)
