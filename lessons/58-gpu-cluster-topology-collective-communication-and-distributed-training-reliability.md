# GPU cluster topology, collective communication, and distributed training reliability

Distributed training is not “one training job, but with more GPUs.” It is a synchronized dataflow system whose speed and correctness depend on where model state lives, how tensors move, which links carry collectives, and how the job survives partial failure.

Adding accelerators can make a job slower when communication, storage, compilation, imbalance, or checkpointing dominates. A reliable design starts with the model-state and communication graph, then maps it onto the physical cluster.

## Mental model: map a communication graph onto a failure topology

Think in two graphs.

**Logical training graph**

```text
data -> forward -> activations -> backward -> gradients
          |                         |
      parameters <- optimizer <- synchronized state
```

**Physical hierarchy**

```text
HBM
 -> GPU-to-GPU link within a node
 -> PCIe / host memory
 -> rack fabric
 -> cross-rack network
 -> checkpoint and dataset storage
```

Every parallelism strategy splits a different object and creates different collectives. The goal is not maximum GPU count; it is minimum time to a reproducible, qualified model under cost and failure constraints.

A useful step-time model is:

[
T_{step} approx max(T_{compute}, T_{exposed communication}) + T_{input} + T_{imbalance} + T_{synchronization}
]

Communication overlapped with useful compute is less harmful than exposed communication on the critical path.

## When to distribute training

Use multiple accelerators when one device cannot meet a concrete requirement:

- model parameters, activations, optimizer state, or sequence length do not fit;
- one-device time to result is unacceptable;
- the batch or data volume needs more throughput;
- a sparse model requires expert distribution;
- training must finish inside a scheduled capacity window.

Prefer one device or one node when it fits and completes within the objective. Single-node jobs are easier to reproduce, debug, checkpoint, and operate.

Scale only after profiling compute, memory, input, and checkpoint behavior. Distributed training cannot fix a slow tokenizer, unsharded object store, tiny kernels, or data-loader contention.

## Account for model-state memory

Estimate memory before choosing parallelism:

- parameters;
- gradients;
- optimizer states;
- master weights for mixed precision;
- activations and temporary workspaces;
- communication buffers;
- compiler/runtime overhead;
- fragmentation;
- checkpoint staging.

Activation memory grows with microbatch size, sequence length, hidden dimensions, layers, and retained intermediates. Activation checkpointing trades extra compute for lower memory by recomputing selected values during backward.

Optimizer-state sharding can divide parameters, gradients, and states across workers, but introduces gather and reduce communication. Leave safety headroom; a job that fits only in the ideal calculation may fail under real fragmentation or variable sequence lengths.

## Parallelism strategies

| Strategy | Splits | Main communication | Best fit | Main risk |
|---|---|---|---|---|
| Data parallelism | Batch/examples | Gradient all-reduce or reduce-scatter | Model fits per worker | Communication grows with model size |
| Sharded data parallelism | Model/gradient/optimizer states | All-gather and reduce-scatter | State does not fit redundantly | Frequent state movement |
| Tensor parallelism | Operations within layers | All-reduce/all-gather per layer | Large dense layers | Latency-sensitive communication |
| Pipeline parallelism | Layer groups/stages | Activations and gradients between stages | Deep models across devices | Bubbles and stage imbalance |
| Sequence/context parallelism | Sequence dimension | Activation exchanges | Very long contexts | Complex attention communication |
| Expert parallelism | Mixture-of-experts experts | Token all-to-all | Sparse expert models | Hot experts and network pressure |
| Hybrid parallelism | Several dimensions | Multiple collective patterns | Very large models | Placement and debugging complexity |

Choose the fewest dimensions that meet memory and time objectives. Every additional dimension adds failure modes and configuration search.

## Collective communication

Common collectives include:

- **all-reduce:** combine values and return the result to every rank;
- **reduce-scatter:** combine and partition the result across ranks;
- **all-gather:** collect partitions so every rank receives the full object;
- **broadcast:** send from one rank to all others;
- **all-to-all:** exchange distinct partitions among all ranks;
- **point-to-point send/receive:** connect pipeline stages.

Collective performance depends on message size, topology, algorithm, congestion, and rank placement. Ring algorithms use bandwidth efficiently for large messages but require multiple steps. Tree-like algorithms can reduce latency for smaller messages. Hierarchical collectives may aggregate within a node before crossing the slower fabric.

Do not assume the communication library always chooses the best path. Benchmark representative message sizes and world sizes on the actual topology.

## Topology-aware placement

A scheduler should understand:

- accelerator type, memory, and supported precision;
- high-bandwidth links inside a node;
- NUMA and CPU affinity;
- NIC locality to GPUs;
- rack and fabric domains;
- oversubscribed network links;
- storage and dataset locality;
- failure domains and maintenance state.

Keep high-frequency tensor-parallel groups on the fastest local fabric. Place pipeline boundaries where activation transfer is affordable. Spread replicas across failure domains only when resilience benefits exceed communication cost.

Rank numbering matters because collective libraries build rings and trees from ranks. Record physical placement and rank topology with each run.

Avoid mixing accelerator generations in a tightly synchronized group unless measured and supported. The slowest rank determines synchronous progress.

## Data pipeline and storage

Each rank needs deterministic, non-overlapping data ownership under the intended sampling policy.

Use:

- immutable dataset manifests and shard hashes;
- rank- and epoch-aware sampling;
- checkpointed sampler position;
- local or node-level caching;
- bounded prefetch;
- resumable streaming;
- explicit handling of corrupt or missing shards.

Prevent every rank from independently reading the same remote objects. That creates storage amplification and thundering herds.

A restart must not silently skip or duplicate data unless the algorithm explicitly allows it. Record global examples/tokens consumed, not only epochs, because elastic world sizes change steps per epoch.

## Stragglers and synchronization

Synchronous jobs advance at the speed of the slowest participating rank. Stragglers can come from:

- thermal or power throttling;
- GPU or memory errors;
- network retransmission or congestion;
- CPU/NUMA misplacement;
- data-loader stalls;
- uneven sequence lengths;
- pipeline-stage imbalance;
- mixture-of-experts token skew;
- background maintenance or noisy neighbors;
- first-use compilation.

Measure per-rank compute, communication, input wait, memory, clock, and collective duration. Aggregate percentiles can hide one bad rank.

Mitigations include bucketing examples by length, balancing pipeline stages, capacity-limiting experts, isolating training networks, pinning CPU/NIC affinity, precompiling, and replacing unhealthy nodes. Do not simply increase collective timeouts; that converts a diagnosable failure into expensive waiting.

## Checkpoint architecture

A checkpoint is a distributed consistency boundary. It may include:

- model parameters and buffers;
- optimizer and scheduler state;
- gradient scaler;
- random-number-generator states;
- data-sampler cursor;
- global step and tokens/examples consumed;
- parallelism topology and shard metadata;
- training configuration, code, container, and dataset manifest;
- privacy or curriculum state;
- validation and run lineage.

Write rank-local shards to durable storage through a two-phase pattern:

1. ranks write uniquely named temporary shards and hashes;
2. a coordinator validates the complete set;
3. the system publishes one immutable manifest as the committed checkpoint.

A checkpoint is valid only when its manifest is committed. Garbage-collect orphaned temporary shards later.

Test restore regularly on spare capacity. A checkpoint that has never been restored is an assumption.

## Resume across topology changes

A checkpoint format tightly coupled to rank count can prevent recovery on a different cluster. Separate logical tensors from physical shard layout when topology portability is required.

Resharding may be expensive and require temporary memory. Validate:

- parameter and optimizer-state mapping;
- RNG and sampler semantics;
- global batch and learning-rate assumptions;
- pipeline partition changes;
- tied weights;
- adapter and expert ownership;
- numerical equivalence within defined tolerance.

Changing world size during training changes optimization unless the effective global batch and schedule are handled deliberately. “Elastic” should describe a tested algorithm and checkpoint path, not merely a scheduler that restarts workers.

## Failure detection and recovery

Distinguish:

- **fail-stop:** worker or node exits;
- **hang:** rank stops progressing while still alive;
- **slow rank:** job progresses at degraded speed;
- **network partition:** subsets cannot complete collectives;
- **silent data corruption:** computation completes with wrong values;
- **storage corruption:** checkpoint or dataset bytes change;
- **control-plane error:** ranks receive inconsistent configuration.

Use heartbeats plus progress counters, collective watchdogs, per-rank telemetry, ECC/error signals, NaN/Inf checks, gradient/activation norms, and artifact hashes.

When one rank fails, abort the entire synchronous communication group unless the framework explicitly supports membership changes. Fence old ranks before restart so a delayed process cannot write into the new run.

Resume from the last committed checkpoint; do not combine partial state from different steps.

## Numerical correctness and reproducibility

Distributed order changes floating-point reduction results. Bitwise identity may be impractical across topology, kernels, or hardware, but the expected tolerance must be defined.

Record:

- random seeds and per-rank RNG state;
- collective and kernel versions;
- precision and loss-scaling policy;
- deterministic-operation settings;
- topology and world size;
- compiler and fusion decisions;
- data ordering.

Test convergence and model-quality equivalence, not only step-level loss. Silent divergence may appear much later.

Use invariant checks for impossible gradients, parameter explosions, missing shards, repeated samples, and cross-rank disagreement.

## Performance and reliability metrics

Track:

- step time distribution and variance;
- samples or tokens per second;
- achieved compute utilization;
- exposed communication time by collective;
- per-rank input wait;
- pipeline bubble fraction;
- expert load distribution;
- GPU memory and fragmentation;
- network throughput, errors, and retransmits;
- checkpoint duration, size, and restore success;
- lost accelerator-hours from failure;
- restart and recovery time;
- validation quality per consumed token and per cost.

High accelerator utilization is not sufficient. The job can efficiently optimize the wrong data, fail to checkpoint, or diverge numerically.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| More data-parallel replicas | Higher throughput | Larger collectives and global-batch changes |
| Tensor parallelism | Fits large layers | Frequent latency-sensitive communication |
| Pipeline parallelism | Fits deep models | Bubbles and stage imbalance |
| Aggressive state sharding | Lower per-rank memory | More all-gather/reduce-scatter traffic |
| Frequent checkpoints | Lower lost work | Storage traffic and training pauses |
| Asynchronous checkpointing | Less foreground pause | Extra memory and consistency complexity |
| Cross-rack placement | More available capacity | Slower, congested communication |
| One-node placement | Fast local links | Smaller scale and node-level blast radius |
| Elastic world size | Better capacity utilization | Optimization and resharding complexity |
| Deterministic kernels | Easier debugging | Potential performance loss |

## Failure modes and defenses

| Failure mode | Defense |
|---|---|
| Adding GPUs makes training slower | Profile exposed collectives and weak scaling on real topology |
| Tensor-parallel ranks cross a slow rack link | Topology-aware group placement |
| One rank reads corrupt data and diverges | Dataset hashes, deterministic shard ownership, cross-rank invariants |
| Checkpoint directory exists but is incomplete | Temporary shards plus atomic committed manifest |
| Restart repeats or skips examples | Persist sampler cursor and global consumed units |
| Slow GPU stalls all workers | Per-rank telemetry, health scoring, and bounded watchdog |
| Collective hang waits for hours | Progress watchdog, diagnostic dump, abort, and fenced restart |
| Old rank rejoins after restart | Run epoch/fencing token on communication and checkpoint writes |
| World-size change silently changes batch | Explicit global-batch, schedule, and convergence policy |
| Mixed hardware creates stragglers | Homogeneous synchronized groups or separately scheduled cohorts |
| Network congestion appears as model slowness | Collective traces, fabric telemetry, and traffic isolation |
| Silent numerical corruption reaches a release | NaN/norm checks, validation checkpoints, redundancy where justified |
| Restore works only with original rank count | Portable manifest and tested resharding path |

## Construction example: multimodal drawing model fine-tuning

A construction AI team fine-tunes a multimodal model on drawing sheets, RFIs, submittals, and site photos. The model fits only with sharded parameters and tensor parallelism, and the job runs across four eight-GPU nodes.

Tensor-parallel groups stay within each node’s high-bandwidth GPU fabric. Data-parallel synchronization crosses nodes. Drawing examples are bucketed by image resolution and token length to reduce stragglers. Each rank receives deterministic dataset shards from an immutable manifest, while node-local caches prevent 32 independent reads of the same large PDFs.

One node develops intermittent network retransmits. Average GPU utilization looks acceptable, but per-rank collective traces identify the affected node. A watchdog aborts the synchronized group, fences the run epoch, and restarts from the last committed checkpoint on a healthy node.

The checkpoint manifest includes model and optimizer shards, RNG states, sampler position, global tokens consumed, topology, dataset hashes, and code/runtime identity. A restore test verifies resharding before the original capacity reservation ends.

A healthcare-operations training workload follows the same reliability design, with stricter access, residency, de-identification, and dataset governance. Distributed training does not permit raw PHI to appear in unrestricted debug dumps or checkpoints.

## Interview prompt

**Design a reliable 32-GPU cluster for fine-tuning a multimodal construction model on drawings, photos, and text. The model does not fit on one GPU, sequence lengths vary, the network is oversubscribed across racks, nodes occasionally fail, and the job must resume without duplicating or skipping data.**

## Worked answer

1. **Clarify workload and objective.** Define model/state size, sequence and image distributions, target time, quality, precision, global batch, hardware inventory, topology, failure rate, storage, and recovery objective.
2. **Calculate memory.** Account for weights, gradients, optimizer, master weights, activations, communication buffers, workspace, and headroom; use activation checkpointing where measured.
3. **Choose minimal parallelism.** Keep tensor-parallel groups inside each eight-GPU node, shard optimizer state as needed, and use data parallelism across nodes. Add pipeline parallelism only if memory still requires it.
4. **Place topology-aware.** Bind GPUs to nearby CPUs and NICs, avoid oversubscribed cross-rack links for frequent collectives, and record rank-to-hardware mapping.
5. **Build deterministic input.** Use immutable manifests, hashed shards, rank-aware sampling, length/resolution buckets, node caches, bounded prefetch, and persisted sampler state.
6. **Optimize collectives.** Benchmark representative all-reduce, all-gather, and reduce-scatter sizes; overlap communication with compute; inspect per-rank traces rather than assuming defaults.
7. **Checkpoint transactionally.** Write sharded temporary files, validate all hashes, commit one manifest, preserve RNG and sampler state, and test restoration and resharding.
8. **Detect failures.** Combine heartbeats, progress counters, watchdogs, ECC and fabric signals, per-rank timing, and numerical invariants. Abort the synchronized group on unsafe membership loss.
9. **Recover safely.** Fence the old run epoch, allocate a compatible topology, restore only a committed checkpoint, verify global batch and data cursor, then run a short validation before continuing.
10. **Operate by outcomes.** Track time and cost per qualified checkpoint, lost accelerator-hours, restore success, exposed communication, stragglers, and final quality—not utilization alone.

**One-line conclusion:** Reliable distributed training maps the model’s communication graph onto the fastest compatible topology, commits complete replayable state, and treats every slow rank, failed collective, and checkpoint as a correctness event—not merely an infrastructure inconvenience.

## References

- [PyTorch Distributed Overview](https://pytorch.org/tutorials/beginner/dist_overview.html)
- [NVIDIA Collective Communications Library documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism](https://arxiv.org/abs/1909.08053)
- [ZeRO: Memory Optimizations Toward Training Trillion Parameter Models](https://arxiv.org/abs/1910.02054)
- [GPipe: Efficient Training of Giant Neural Networks using Pipeline Parallelism](https://proceedings.neurips.cc/paper/2019/hash/093f65e080a295f8076b1c5722a46aa2-Abstract.html)
- [NVIDIA GPUDirect RDMA documentation](https://docs.nvidia.com/cuda/gpudirect-rdma/)
- [MLPerf Training benchmark rules and results](https://mlcommons.org/benchmarks/training/)
