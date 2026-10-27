# 34 — AI inference serving, batching, caching, and GPU capacity

Previous: [Load testing, chaos experiments, and performance validation](33-load-testing-chaos-experiments-and-performance-validation.md). Study time: 55–65 minutes including the exercise.

## Mental model

LLM inference is a scheduling system for variable-sized work competing for scarce memory and compute.

```text
request
  → policy and token estimate
  → admission / queue / model route
  → prefill
  → iterative decode
  → stream or structured result
  → validation and workflow decision
```

> Optimize for useful, policy-compliant outcomes per unit of scarce capacity—not raw requests per second or GPU utilization alone.

Two phases behave differently:

- **Prefill** processes the input prompt, is highly parallel, and its cost grows with input length.
- **Decode** generates tokens iteratively, repeatedly reading model weights and KV-cache state; sequence length and concurrent active sequences matter.

A 500-token extraction and a 100,000-token drawing-set summary are not equivalent requests. Capacity, queues, quotas, and cost must use tokens and memory—not only request count.

## When to use dedicated inference infrastructure

Use a managed model API when speed of delivery, elastic capacity, and reduced operations outweigh the need for low-level control.

Consider self-hosted or dedicated serving when requirements include:

- model weights or fine-tuned adapters you control;
- data residency or network isolation;
- predictable high sustained volume;
- custom batching, quantization, or latency policy;
- approved hardware/region constraints;
- strict tenant isolation;
- provider quota independence;
- detailed token-level observability.

Self-hosting adds model artifact security, GPU scheduling, driver/runtime compatibility, capacity planning, rollout, evaluation, and incident ownership. Do not self-host solely because GPU hourly cost looks lower; include idle capacity, redundancy, engineering, storage, network, and failure recovery.

## Define service classes

Separate work before selecting hardware:

| Class | Typical objective | Scheduling treatment |
| --- | --- | --- |
| Interactive Q&A | Low time to first token | Reserved concurrency, short queue |
| Structured extraction | Correct bounded completion | Batchable, schema validation |
| Agent/tool planning | Predictable validated result | Strict output/tool limits |
| Long summarization | Completion throughput | Async queue or batch |
| Embedding/reranking | High item throughput | Independent serving pool |
| Evaluation/backfill | Low priority and interruptible | Batch/off-peak capacity |
| Safety/reconciliation support | High correctness/availability | Protected capacity |

Mixing every class in one FIFO queue creates head-of-line blocking and makes SLOs impossible to reason about.

## Token and memory accounting

Estimate at admission:

```text
input tokens
+ reserved output tokens
+ model/runtime overhead
+ KV-cache requirement
+ adapter/context overhead
```

KV-cache memory grows with active sequence count and token length. Exact bytes depend on architecture, data type, parallelism, and runtime, so benchmark the real stack rather than relying on one universal formula.

Important limits:

- model weights must fit across the chosen devices;
- KV cache must fit for admitted sequences;
- temporary activations and runtime buffers need headroom;
- long contexts can evict or block many shorter requests;
- fragmentation can reduce usable memory below reported free memory.

Reject, defer, shorten with user consent, or route before allocating impossible work. An out-of-memory crash is a poor admission policy.

## Batching

### Static batching

Collect a fixed batch, run it together, and wait for the batch to finish.

Good for offline work with similar lengths. Poor for interactive generation because one long sequence delays shorter ones and completed slots remain unused.

### Dynamic batching

Wait briefly to form a batch from requests arriving near one another.

Improves accelerator utilization but adds queue delay. The batch window must fit the request class’s latency budget.

### Continuous or in-flight batching

The scheduler adds new sequences as others finish while decode continues.

Benefits:

- better utilization under variable output lengths;
- lower head-of-line blocking;
- more completed tokens per unit time.

Costs:

- complex scheduling and memory management;
- latency can degrade under aggressive concurrency;
- fairness must be explicit.

Batching is not automatically beneficial. Measure TTFT, inter-token latency, completion latency, throughput, and quality under the actual length distribution.

## Scheduling policies

A production scheduler considers:

- request class and priority;
- arrival time and deadline;
- input length;
- maximum/reserved output length;
- KV-cache footprint;
- tenant quota;
- model/adapter;
- batching compatibility;
- preemption/recompute cost;
- fairness and starvation.

FIFO is simple but a few long prompts can block interactive work. Shortest-estimated-job-first improves average latency but can starve large requests and depends on estimates. Priority scheduling protects urgent work but needs reserved capacity and aging.

Use separate lanes or pools for interactive and batch work. Limit each tenant’s active tokens or sequences so one customer cannot fill the entire KV cache.

## Backpressure and admission control

Admission should happen before expensive tokenization, retrieval fan-out, or GPU allocation where possible.

Decide from:

- estimated tokens and memory;
- current active sequences;
- queue age and predicted completion;
- request deadline;
- tenant budget;
- model health;
- approved fallback;
- downstream tool capacity;
- safety/risk class.

Possible outcomes:

- admit now;
- enqueue with operation ID;
- route to another compatible deployment;
- use an evaluated smaller model;
- reduce optional work;
- return evidence-only output;
- request narrower scope;
- reject with a retry hint.

A queue does not create GPU capacity. Bound it by work units and deadlines, and expire work that can no longer provide value.

## Autoscaling GPU inference

CPU-style utilization is often a weak scaling signal. GPU utilization can be high while TTFT is acceptable, or low while KV cache is exhausted.

Useful signals include:

- queued input and reserved output tokens;
- oldest queue age;
- active sequences;
- KV-cache occupancy;
- prefill/decode utilization;
- TTFT and inter-token latency;
- admission rejection;
- model-specific throughput.

Scale-out delay includes node provisioning, image pull, driver/runtime startup, model weight loading, compilation, cache warming, health checks, and discovery propagation. Keep warm capacity for interactive SLOs and known bursts.

Scale-in requires draining. Stop admitting new work, finish or checkpoint bounded requests, handle streams, and release model memory. Killing a replica with active tool-planning output can create incomplete workflows; deterministic workflow state must decide whether to retry.

## Parallelism choices

Large models may require:

- **tensor parallelism:** split matrix computation across devices;
- **pipeline parallelism:** split layers/stages;
- **data parallelism:** replicate model instances;
- **expert parallelism:** distribute mixture-of-experts components.

More devices can fit a model or improve throughput, but communication and synchronization grow. Data-parallel replicas often isolate failures and simplify scaling when one replica can fit the model. Multi-device replicas increase the failure domain and make fragmentation/capacity placement harder.

Benchmark the exact topology, interconnect, model, precision, lengths, and runtime.

## Quantization

Lower-precision weights—and sometimes activations or KV cache—can reduce memory and improve throughput.

Tradeoffs:

- accuracy and calibration changes;
- hardware/kernel compatibility;
- differing gains by model and workload;
- potential degradation in rare or safety-critical slices;
- operational complexity across model variants.

Do not approve quantization from aggregate benchmark scores alone. Re-run task, language, document-quality, citation, structured-output, and safety evaluations. Version quantization configuration as part of the model artifact.

## Speculative decoding

A smaller draft model proposes tokens; the target model verifies them. If acceptance is high, generation can be faster without changing the target distribution under the algorithm’s assumptions.

Benefits depend on draft quality, target model, workload, batch size, and hardware. Draft computation and coordination can hurt when acceptance is low.

Treat speculative configuration as a serving-version change: benchmark latency/cost and revalidate outputs and streaming behavior.

## Prefix and KV reuse

Repeated system prompts, tenant policy prefixes, or shared document prefixes may allow prefix/KV reuse.

Benefits:

- lower repeated prefill work;
- improved TTFT.

Risks:

- memory pressure;
- invalid reuse after prompt/policy/model change;
- cross-tenant leakage;
- authorization changes;
- low hit rate from minor prompt differences.

A safe key includes model and tokenizer version, exact prefix hash, tenant/security scope, policy and prompt version, adapter, and relevant data version. Never share sensitive cached state across tenants merely because text hashes match.

## Response and semantic caching

### Exact response cache

Return a prior response for an identical normalized request and dependency version.

Use for deterministic, low-risk work when freshness and authorization are explicit.

### Semantic cache

Reuse an answer for a sufficiently similar query.

It can save cost but similarity does not prove equivalent intent, authorization, or required freshness. Risks include wrong answers, stale citations, cross-tenant exposure, and bypassed policy.

For construction or healthcare evidence, cache derived results only with:

- tenant and authorization scope;
- source/index version;
- model/prompt/policy version;
- freshness budget;
- risk classification;
- traceable citations;
- invalidation/tombstone handling.

Do not semantic-cache consequential tool decisions.

## Model routing

A gateway may route by:

- task and modality;
- risk and required capability;
- evaluated quality;
- context length;
- region/privacy;
- tenant contract;
- latency/deadline;
- quota and capacity;
- cost.

Eligibility is a hard filter; performance and cost optimize only among eligible models.

Record model, provider, deployment, artifact, tokenizer, quantization, adapter, prompt, policy, retrieval index, and routing reason. A fallback is a new evaluated configuration, not simply another endpoint.

## Streaming and cancellation

Streaming improves perceived latency but introduces lifecycle issues:

- client disconnect may not immediately cancel server compute;
- partial output is not a validated result;
- retries can duplicate visible text;
- tool JSON may be incomplete;
- usage accounting must include generated/discarded tokens.

Propagate cancellation and deadlines, but make workflow effects independent of connection state. Execute a tool only after complete structured validation, policy, authorization, and durable state transition.

If a stream fails, label it incomplete. Do not silently concatenate output from another model invocation.

## Multi-tenancy and isolation

Protect inference with:

- trusted tenant identity;
- per-tenant token/concurrency/queue budgets;
- reserved capacity for critical classes;
- scoped caches and adapters;
- residency-aware routing;
- encrypted transport and artifact access;
- prompt/log minimization;
- fair scheduling and aging;
- audited overrides.

LoRA or other tenant adapters reduce per-tenant weight duplication but add adapter loading, cache, version, and isolation concerns. Never choose an adapter from an untrusted request field without authorization.

One tenant’s long-context backfill must not consume all KV cache or force every interactive tenant into a slow fallback.

## Failure handling

Plan for:

- replica crash or GPU error;
- OOM during prefill/decode;
- node loss in a multi-GPU replica;
- corrupt or unavailable model artifact;
- cold-start failure;
- queue/control-plane outage;
- stale routing after drain;
- provider rate limit;
- mid-stream disconnect;
- malformed structured output;
- model-quality regression;
- cache contamination.

An OOM request should be rejected or rerouted based on a stable operation/request ID; uncontrolled retry to every replica creates a fleet-wide failure.

If a model call in a durable workflow fails, replay only the model step when its inputs and policy version are still valid. Never replay an external tool side effect just because generation restarted.

## Construction operations example

A construction assistant serves interactive drawing/RFI questions, invoice extraction, change-order summaries, and a 100,000-page embedding/backfill workload. Month-end produces 5× interactive traffic.

Design:

1. separate interactive generation, structured extraction, embeddings, and offline summaries into lanes;
2. admit by input plus reserved output tokens and KV footprint;
3. reserve interactive and ERP-reconciliation capacity;
4. use continuous batching for variable generation lengths;
5. keep backfills in async queues with lower priority and aging;
6. route only among evaluated models that meet tenant and task requirements;
7. use prefix caching keyed by tenant, model, prompt, policy, and version;
8. exact-cache immutable cited summaries only while source/index versions match;
9. forbid semantic-cache reuse for invoice approval or tool execution;
10. autoscale from queue tokens, queue age, active sequences, KV occupancy, and TTFT;
11. maintain warm replicas for expected month-end bursts;
12. drain streams and stop admissions before scale-in or rollout;
13. record full lineage for every answer/action proposal;
14. re-run extraction, citation, and safety evaluations after quantization or runtime changes.

A long drawing query can be deferred or narrowed rather than allowed to evict hundreds of short interactive sequences.

## Healthcare operations variation

A healthcare operations gateway filters models by PHI approval, region, task, and organization before considering latency or cost. Caches, prefixes, adapters, and logs are organization-scoped and minimized.

Interactive operational reads have protected capacity. Historical summarization and evaluation jobs use separate queues. A smaller fallback model is eligible only if it has passed the same task-specific privacy, quality, and safety gates.

Structured clinical or scheduling outputs require schema and source-version validation. No action follows partial streaming output. During overload, the system may return authorized source evidence or follow the approved manual path rather than route PHI to an unapproved provider.

Quantization, adapter, runtime, or batching changes receive evaluation gates for rare conditions, language, malformed records, hallucination, and authorization-sensitive retrieval.

## Tradeoffs

| Choice | Benefit | Cost or risk |
| --- | --- | --- |
| Managed API | Fast delivery and elasticity | Quota, cost, and control limits |
| Self-hosting | Runtime/data control | GPU and operations burden |
| Static batch | Efficient offline work | Head-of-line blocking |
| Continuous batch | Better mixed-length utilization | Scheduler complexity |
| High concurrency | Throughput | TTFT/inter-token degradation |
| Separate pools | SLO isolation | Stranded capacity |
| One shared pool | Utilization | Noisy-neighbor risk |
| Warm capacity | Fast burst response | Idle cost |
| Scale to zero | Cost savings | Cold-start delay |
| Tensor parallelism | Fits larger models | Communication and failure domain |
| Quantization | Memory/throughput gain | Quality risk |
| Speculative decoding | Potential lower latency | Draft overhead |
| Prefix caching | Lower prefill cost | Scope/invalidation risk |
| Semantic cache | Cost/latency reduction | Incorrect or stale reuse |
| Smaller fallback | Availability/cost | Capability and quality change |
| Longer context | More evidence | KV memory and latency |

## Failure modes

| Failure | Consequence | Defense |
| --- | --- | --- |
| Limits count requests only | Long prompts dominate | Token/KV-weighted admission |
| Reserve no output budget | Decode exhausts memory | Bounded output reservation |
| FIFO mixes batch and interactive | Head-of-line blocking | Class-aware queues/pools |
| GPU utilization alone scales | Late or unstable scaling | Queue/KV/latency signals |
| Scale-out ignores model load time | SLO missed during burst | Warm capacity and measured startup |
| Scale-in kills streams | Partial responses/workflow confusion | Drain and resumable state |
| Aggressive batching | TTFT rises | Class-specific batch policy |
| OOM retries across fleet | Cascading failure | Admission and bounded reroute |
| Long context monopolizes KV | Small requests starve | Per-request/tenant limits |
| Cache omits tenant scope | Data leak | Security-scoped keys |
| Cache omits source version | Stale evidence | Versioned key/invalidation |
| Semantic cache serves tool decision | Unsafe reused action | Never cache consequential intent |
| Fallback ignores residency | Privacy breach | Hard eligibility filter |
| Quantization ships on aggregate score | Rare-slice regression | Risk-based evaluation |
| Partial stream triggers tool | Invalid external effect | Complete validation and durable transition |
| Client disconnect keeps compute | Wasted capacity | Cancellation propagation |
| Shared adapter chosen by request | Tenant crossover | Authorized adapter mapping |
| Backfill consumes every GPU | Interactive outage | Quotas and reserved pools |
| Autoscaling chases short spike | Cost/churn | Hysteresis and queue policy |

## What to measure

Track offered/admitted/rejected/deferred requests; input, reserved output, and generated tokens; queue tokens and age; active sequences; batch size; prefill and decode time; TTFT; inter-token latency; completion latency; tokens/s; KV-cache occupancy and eviction/recompute; GPU memory/utilization/power; OOMs; cold-start stages; autoscaling and drain time; cancellations; cache hit rate and scoped invalidations; per-tenant fairness; model route/fallback; cost per correct outcome; structured-output validity; citations; tool success; safety; and quality by risk slice.

Measure useful outcomes, not just tokens. An answer that is fast but uncited, truncated, or unsafe is not successful capacity.

## Interview prompt

> Design multi-tenant LLM inference serving for a construction AI platform. It supports interactive cited Q&A, invoice extraction, tool planning, offline summaries, and embedding backfills. Month-end traffic is 5×, one tenant sends very long prompts, GPUs are slow to start, and healthcare customers require PHI-approved regional models.

Cover prefill/decode, token and KV admission, batching, scheduling, autoscaling, parallelism, quantization, caching, model routing, streaming, multi-tenancy, failures, evaluation, cost, and healthcare adaptation.

## Worked answer

**One-line conclusion:** Admit and schedule inference by tokens, memory, deadline, and policy; isolate interactive from batch work; and optimize batching, caching, and model variants only behind quality, privacy, and lineage gates.

“I would classify traffic into interactive generation, structured extraction/tool planning, embeddings, and offline batch. Each class has a deadline, token/output bound, quality gate, and reserved capacity. Admission estimates input tokens, maximum output, and KV footprint; impossible or expired work is narrowed, deferred, safely rerouted, or rejected before GPU allocation.

Interactive generation uses continuous batching with tenant concurrency and active-token limits. Batch summaries and backfills use separate lower-priority queues with aging. Autoscaling uses queued tokens, oldest queue age, active sequences, KV occupancy, and TTFT—not GPU utilization alone. Warm replicas cover burst and startup time; scale-in drains streams and active work.

The gateway first filters models by task capability, evaluation approval, privacy, residency, context, and tenant contract, then optimizes latency/cost. Quantized, speculative, adapter, and runtime variants are distinct versioned deployments with risk-slice evaluation.

Prefix and exact response caches include tenant/security scope plus model, tokenizer, prompt, policy, adapter, source, and index versions. Semantic caching is limited to low-risk evidence responses and never reuses tool intent.

Streaming supports cancellation, but partial output cannot commit workflow state. Tool execution follows complete schema validation, current authorization/policy, approval binding, and an idempotent durable operation. I would monitor token queues, TTFT, inter-token latency, KV pressure, OOMs, fairness, cold starts, routes, cache safety, cost, citations, structured validity, and task quality.”

## Practice lab

Design inference capacity for:

- interactive Q&A and tool planning;
- invoice extraction;
- 100,000-page embedding/backfill work;
- 5× month-end arrivals;
- one tenant with 80,000-token prompts;
- slow GPU provisioning and model loading;
- a quantized fallback candidate;
- one replica OOM;
- mid-stream client disconnect;
- PHI requiring a regional approved model.

Specify service classes, token/KV admission, queue bounds, scheduling, batching, warm capacity, autoscaling signals, cache keys, fallback eligibility, drain/retry behavior, evaluation gates, fairness, and metrics.

Then explain what you would degrade first and what you would never degrade.

## References

- [vLLM — Easy, Fast, and Cheap LLM Serving with PagedAttention](https://arxiv.org/abs/2309.06180)
- [Orca — A Distributed Serving System for Transformer-Based Generative Models](https://www.usenix.org/conference/osdi22/presentation/yu)
- [NVIDIA TensorRT-LLM documentation](https://nvidia.github.io/TensorRT-LLM/)
- [Hugging Face Text Generation Inference documentation](https://huggingface.co/docs/text-generation-inference/)
- [KServe documentation](https://kserve.github.io/website/)
