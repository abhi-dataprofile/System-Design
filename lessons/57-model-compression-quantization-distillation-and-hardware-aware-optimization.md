# Model compression, quantization, distillation, and hardware-aware optimization

A smaller model artifact is not automatically a faster, cheaper, or safer system. Compression changes numerical behavior, memory traffic, kernel eligibility, batching, calibration, and sometimes model capability. A theoretically smaller model can run slower if the target hardware lacks efficient kernels or spends time converting formats.

Treat compression as a versioned release for a specific workload and hardware matrix—not as a file-size operation.

## Mental model: optimize the deployed path, not parameter count

The useful object is a complete deployment candidate:

```text
teacher / source model
  -> compression recipe
  -> compressed weights + runtime graph
  -> target compiler and kernels
  -> hardware-specific execution plan
  -> workload benchmark + quality evaluation
  -> signed release by device / accelerator class
```

Evaluate three layers:

1. **Representation:** precision, sparsity, architecture, adapters, and tokenizer.
2. **Execution:** graph transformations, kernels, memory layout, compiler, runtime, and hardware.
3. **Outcome:** task quality, calibration, abstention, latency, throughput, memory, energy, and cost.

The optimization objective is constrained:

[
min 	ext{latency, memory, energy, or cost}
]

subject to quality, safety, compatibility, and reliability gates on the target workload.

## When to compress

Consider compression when:

- weights or KV cache do not fit the required accelerator or edge device;
- memory bandwidth limits inference;
- latency or energy prevents an interactive or mobile use case;
- the model must run offline or in a restricted environment;
- sustained demand justifies hardware-specific optimization;
- a smaller student can handle a bounded workload;
- serving more concurrent sequences creates measurable business value.

Do not compress merely because a lower bit width is available. First profile the system. The bottleneck may be tokenization, retrieval, network, preprocessing, CPU–GPU transfer, queueing, or an external tool.

Prefer routing to an already evaluated small model when it meets the need. Prefer retrieval or deterministic code when the task is knowledge lookup or exact arithmetic. Compression is one option in a system portfolio.

## Build a deployment contract

Before optimization, define:

- target tasks and input distributions;
- prompt, image, and sequence-length distributions;
- batch and concurrency expectations;
- target devices, accelerators, drivers, operating systems, and instruction sets;
- latency metric and percentile;
- throughput under the intended scheduler;
- memory ceiling, including weights, activations, workspace, and KV cache;
- power or energy constraints where measurable;
- quality, calibration, safety, and cohort non-regression gates;
- supported context length and output structure;
- fallback and rollback behavior.

Benchmark cold start, warm steady state, and sustained load. Edge devices may throttle thermally; servers may behave differently under continuous batching.

## Quantization

Quantization represents weights or activations with fewer bits or restricted numeric formats.

### Common choices

| Technique | What changes | Typical implication |
|---|---|---|
| FP16/BF16 | Lower floating-point precision | Broad accelerator support; baseline for many deployments |
| INT8 weights and activations | Integer arithmetic for both | Strong kernel potential; activation calibration is important |
| Weight-only INT8/INT4 | Compress weights; activations remain higher precision | Useful for memory-bound inference; kernel-dependent |
| Mixed precision | Sensitive layers use more bits | Better quality–efficiency tradeoff, more complexity |
| KV-cache quantization | Compress attention cache | Longer context or concurrency, possible long-context degradation |
| Dynamic quantization | Activation ranges chosen at runtime | Flexible, but runtime overhead and variable behavior |
| Static quantization | Ranges calibrated ahead of time | Faster execution where supported; calibration-sensitive |

### Post-training quantization

Post-training quantization converts a trained model using a calibration set, without full retraining. It is fast and often sufficient, but outliers and underrepresented inputs can cause severe local regressions.

The calibration set should cover:

- real prompt and sequence-length distributions;
- languages, document types, and image conditions;
- rare but important classes;
- structured-output and tool-use cases;
- safety and abstention boundaries;
- target tenants or cohorts where permitted.

Never calibrate only on a convenient public benchmark when production inputs are invoices, drawings, or low-light field images.

### Quantization-aware training

Quantization-aware training simulates numerical effects during training or fine-tuning so the model can adapt. It may recover quality, but it adds training complexity, data governance, and a new source-to-candidate lineage.

Track exactly which fake-quantization scheme, scale granularity, rounding, clipping, and target kernels were assumed. A QAT model trained for one runtime may not behave identically in another.

### Granularity and outliers

Scales may be per-tensor, per-channel, per-group, or token-dependent. Finer granularity can preserve quality but costs metadata and kernel complexity.

Outlier channels can dominate scale ranges. Techniques that isolate, rescale, or preserve sensitive channels may help, but must be validated by task and hardware. There is no universal safe bit width.

## Distillation

Knowledge distillation trains a smaller student to reproduce selected behavior of a teacher.

Possible targets include:

- output probabilities or logits;
- intermediate representations;
- embeddings;
- rationales or synthetic demonstrations;
- preference rankings;
- tool-selection and structured-output behavior.

Teacher output is not ground truth. A student can inherit hallucinations, bias, unsafe shortcuts, and calibration errors. Combine teacher supervision with trusted labels and explicit policy tests.

Design the distillation curriculum around the intended workload. A student optimized for invoice extraction should not be advertised as a general replacement for the teacher.

Measure teacher–student disagreement and retain a policy for escalation. Distillation pairs naturally with routing: the student handles the evaluated domain, while uncertain or unsupported cases go to a larger eligible model or human review.

## Pruning and sparsity

Pruning removes weights, channels, heads, layers, or blocks.

- **Unstructured sparsity** zeros individual weights; file size may fall, but speed improves only with compatible sparse kernels and enough sparsity.
- **Structured sparsity** removes hardware-friendly groups, channels, heads, or blocks; it is easier to accelerate but can damage capacity.
- **Depth reduction** removes layers and changes latency predictably, but may affect reasoning and long-context behavior.
- **Dynamic sparsity or conditional computation** activates parts of the model per input; it adds routing and load-balancing complexity.

Report physical runtime gains, not just percentage of zero weights.

## Architecture and graph optimization

Hardware-aware optimization may include:

- operator fusion;
- constant folding and dead-node elimination;
- attention and normalization kernel selection;
- static or bounded shapes;
- memory planning and buffer reuse;
- layout conversion;
- speculative decoding;
- tensor, pipeline, or sequence parallelism;
- compiler autotuning;
- adapter merging;
- tokenizer and preprocessing optimization.

Graph transformations can alter numerical order. Verify outputs and quality, especially for thresholds, structured schemas, and long sequences.

Pin compiler, driver, runtime, kernel library, and hardware firmware versions in the release manifest.

## KV cache and long context

For autoregressive models, KV cache can dominate memory as context and concurrency grow. Options include:

- lower-precision KV storage;
- paged allocation;
- prefix caching;
- context truncation or retrieval;
- grouped-query or multi-query attention in an alternative architecture;
- admission limits by estimated cache usage.

Evaluate long-context retrieval, recency, copy accuracy, multilingual behavior, and interruption/reconnect paths. A model that passes short prompts may fail after cache quantization at production context lengths.

Do not share prefix or KV caches across tenants unless content, authorization, policy, and access-pattern risks are explicitly controlled.

## Hardware-aware benchmarking

Benchmark on the exact supported matrix. Record:

- hardware model and memory;
- driver, firmware, compiler, runtime, and kernel versions;
- precision and quantization recipe;
- batch, concurrency, input length, output length, and decoding policy;
- cold-start and steady-state latency distributions;
- prefill and decode throughput separately;
- peak and sustained memory;
- power/energy measurement method;
- thermal state for edge devices;
- errors, fallback, and unsupported-operator frequency.

Synchronize asynchronous devices before measuring. Exclude warm-up only if production has an equivalent warm path, and report both cold and warm behavior.

An optimized benchmark that omits tokenization, image preprocessing, transfer, or validation may be useful for kernel analysis but is not end-to-end latency.

## Quality and safety evaluation

Run the same release gates as a model change:

- representative task accuracy and utility;
- calibration and abstention;
- exact structured output and schema adherence;
- tool selection and argument correctness;
- citation and grounding behavior;
- long-context and multilingual suites;
- rare and high-impact cohorts;
- adversarial and safety tests;
- fairness and tenant slices;
- teacher/source disagreement analysis.

Set non-regression thresholds before viewing results. Compression errors may cluster around rare tokens, small numeric differences, low-contrast images, or borderline confidence thresholds.

For exact calculations, identifiers, or totals, use deterministic validation rather than trusting approximate arithmetic at any precision.

## Release and routing

Register each compressed artifact separately by:

- source model and license;
- compression code and recipe;
- calibration/training data lineage;
- compiler and runtime;
- target hardware eligibility;
- evaluation and benchmark evidence;
- known limitations;
- rollback artifact.

Route only to compatible hardware. If the preferred kernel is unavailable, fail to a known evaluated path rather than silently dequantizing or using an unbenchmarked fallback.

Use shadowing, canaries, and cohort-aware rollout. Monitor quality proxies, fallback rate, memory pressure, throttling, latency, and hardware-specific errors.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Post-training quantization | Fast and inexpensive | Calibration-sensitive regressions |
| Quantization-aware training | Better adaptation to low precision | Training, data, and lineage complexity |
| Weight-only quantization | Lower weight memory and bandwidth | Activations/KV remain large |
| Weight-and-activation quantization | Greater kernel and memory gains | Higher quality sensitivity |
| Distillation | Smaller purpose-built student | Teacher errors and narrower capability |
| Unstructured pruning | Flexible compression | Often no speedup without sparse kernels |
| Structured pruning | Hardware-friendly reduction | More capacity loss |
| Hardware-specific compilation | Maximum performance | Portability and fleet fragmentation |
| One universal artifact | Simpler operations | Leaves performance unused or fails on some devices |
| Artifact per hardware class | Better optimization | More evaluation, registry, and rollout work |

## Failure modes and defenses

| Failure mode | Defense |
|---|---|
| Model file shrinks but latency increases | Profile target kernels, conversions, and end-to-end path |
| Calibration set misses handwritten invoices | Production-representative, rare-case, and cohort calibration |
| INT4 model fails only on long contexts | Context-length and KV-cache evaluation matrix |
| Sparse weights receive no speedup | Verify supported sparse pattern and physical kernel metrics |
| Student copies teacher hallucinations | Trusted labels, disagreement analysis, grounding and safety gates |
| Compiler silently falls back to slow operators | Fail build or expose fallback metrics; benchmark actual execution plan |
| Different driver changes behavior | Pin and qualify the complete hardware/runtime matrix |
| Quantization changes confidence thresholds | Recalibrate and reevaluate abstention and routing policies |
| Edge device passes cold test then throttles | Sustained thermal and battery testing |
| Compressed model breaks JSON or tool arguments | Exact schema, argument, and integration tests |
| Optimized cache leaks between tenants | Tenant-scoped cache keys, isolation, and access-pattern review |
| Rollback artifact is incompatible with new runtime | Preserve complete compatible release bundles |
| Average quality holds but small cohort collapses | Worst-cohort and non-regression release gates |

## Construction example: field inspection assistant

A construction company wants a vision-and-language assistant on tablets used in basements with unreliable connectivity. The source model is too large for the device and drains the battery.

Profiling shows image encoding and model weights dominate latency; network is not the relevant bottleneck offline. The team distills a student for bounded tasks: classify inspection photos, extract room and asset labels, and draft a finding. It applies weight-only quantization first because the device runtime has optimized kernels.

The calibration and evaluation sets include low light, dust, occlusion, handwritten labels, multiple project types, rare firestopping defects, and exact room identifiers. The compressed model meets average accuracy but frequently changes one digit in identifiers, so deterministic OCR validation and a confirmation step remain mandatory.

A second hardware class lacks the required INT4 kernel and becomes slower through conversion. The registry therefore publishes separate device manifests: an INT4 artifact for supported tablets and an INT8 artifact for the older fleet. Unsupported devices use a bounded cloud route when connected or a simpler offline checklist.

A healthcare-operations version could optimize room-turnover or equipment-inventory models, but device-specific quality, protected cohorts, PHI handling, sustained thermal behavior, and abstention remain release gates. Compression never expands the model’s approved clinical role.

## Interview prompt

**Design a compression and deployment pipeline for a construction field assistant that must run offline on two tablet classes. It processes photos and text, has strict memory and battery limits, must preserve rare safety cases and exact identifiers, and can fall back to cloud inference only when connected.**

## Worked answer

1. **Clarify the contract.** Define tasks, device matrix, workload distributions, memory, latency, energy, context, quality, safety, offline behavior, and cloud eligibility.
2. **Profile first.** Measure preprocessing, transfer, weights, activations, KV cache, kernels, and sustained thermal behavior to identify the real bottleneck.
3. **Choose bounded techniques.** Distill a task-specific student, then compare FP16/BF16, INT8, and supported weight-only INT4; use QAT only if PTQ misses gates.
4. **Build representative data.** Version calibration, distillation, and evaluation sets across lighting, layouts, identifiers, projects, rare hazards, languages, and sequence lengths.
5. **Compile per hardware class.** Pin graph transformations, compiler, runtime, drivers, kernels, shapes, and fallback behavior; reject unsupported operators.
6. **Evaluate outcomes.** Test task utility, exact identifiers, structured output, calibration, abstention, safety, long context, cohorts, and source-model disagreement.
7. **Benchmark honestly.** Report cold/warm and sustained end-to-end latency, throughput, peak memory, battery/energy method, throttling, and fallback rates on real devices.
8. **Register complete artifacts.** Store source, recipe, data lineage, target matrix, execution plan, evidence, limitations, and compatible rollback bundle.
9. **Route safely.** Select only artifacts qualified for the device; use deterministic validation and abstain or use an approved cloud/manual path when unsupported.
10. **Roll out progressively.** Shadow where possible, canary by device class, monitor quality and hardware errors, and atomically revert the full release when gates fail.

**One-line conclusion:** Model compression succeeds only when a hardware-qualified release reduces measured end-to-end resource use while preserving task, cohort, safety, and fallback guarantees on the real workload.

## References

- [Distilling the Knowledge in a Neural Network](https://arxiv.org/abs/1503.02531)
- [Deep Compression: Compressing Deep Neural Networks with Pruning, Trained Quantization and Huffman Coding](https://arxiv.org/abs/1510.00149)
- [GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers](https://arxiv.org/abs/2210.17323)
- [AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration](https://proceedings.mlsys.org/paper_files/paper/2024/hash/42a452cbafa9dd64e9ba4aa95cc1ef21-Abstract-Conference.html)
- [ONNX Runtime quantization documentation](https://onnxruntime.ai/docs/performance/model-optimizations/quantization.html)
- [NVIDIA TensorRT documentation](https://docs.nvidia.com/deeplearning/tensorrt/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
