# AI cost governance, unit economics, and capacity portfolios

AI cost governance is not a monthly cloud-bill review. It is a control system that connects product outcomes to variable model, infrastructure, data, and human-review costs while preserving quality, safety, and contractual obligations.

A cheap model that creates more rework can be more expensive than a premium model. A highly utilized GPU fleet can still waste money if it serves low-value traffic while urgent work waits.

## Mental model: spend flows through capability to outcome

Model the system as a chain:

```text
business demand
  -> workload class
  -> model / tool / human route
  -> metered resources
  -> quality and operational outcome
  -> attributed unit economics
  -> forecast, budget, and policy feedback
```

Track three layers separately:

1. **Resource cost:** provider tokens, GPU-seconds, CPU, storage, network, vector search, third-party tools, and reviewer time.
2. **Service unit:** document processed, conversation resolved, drawing sheet indexed, RFI drafted, or inspection reviewed.
3. **Business outcome:** accepted extraction, avoided rework, cycle time, recovered revenue, or safe abstention.

The key equation is not just cost per request:

[
	ext{quality-adjusted cost per accepted outcome}
=
rac{	ext{AI + infrastructure + review + rework cost}}
{	ext{accepted outcomes}}
]

Use measured values and confidence intervals. Do not invent savings or treat every generated token as value.

## When to use a cost-governance layer

Add explicit governance when:

- multiple teams or tenants share models and accelerators;
- provider prices, token lengths, or traffic vary materially;
- workloads can route among models, batch jobs, edge devices, or human review;
- reserved capacity, owned infrastructure, and on-demand APIs coexist;
- a runaway agent or retry loop can create significant spend;
- contracts require quotas, cost attribution, or predictable service levels.

For a small prototype, simple provider billing and request counters may be sufficient. Introduce the control plane before production scale, not after an unexpected invoice.

## Define a cost taxonomy

Classify cost before allocating it:

| Cost class | Examples | Allocation approach |
|---|---|---|
| Direct variable | Input/output tokens, image pages, tool calls | Attribute to request and workload |
| Shared variable | GPU pool, vector cluster, review queue | Allocate by measured consumption drivers |
| Fixed committed | Reserved capacity, licenses, minimum spend | Assign to portfolio, then report utilized and stranded portions |
| Platform overhead | Gateways, observability, security, evaluation | Allocate transparently or report separately |
| Failure and rework | Retries, invalid outputs, corrections, escalations | Charge to the originating workflow, not hide as “operations” |
| Opportunity cost | Urgent work delayed by low-value traffic | Model through SLO impact and admission decisions |

Never allocate shared GPU cost only by request count. A long-context generation and a short classifier consume different resources. Capture prompt tokens, generated tokens, images, batch size, active duration, accelerator class, cache hits, and queue priority.

## Metering and attribution

Emit an immutable usage record after every attempt, including failed and cancelled attempts:

- request, trace, tenant, product, environment, and workload-class IDs;
- model/provider/version and route reason;
- input/output units and billable tool units;
- cached versus newly computed work;
- region, accelerator type, runtime duration, and batch share;
- retry, cancellation, fallback, and error reason;
- review minutes and final outcome when known;
- price-card and allocation-policy versions.

Keep provider invoices as reconciliation data, not the only source of truth. Price lists change and providers may round or aggregate differently. Store effective dates and currencies, and distinguish observed cost from estimated cost.

Use high-cardinality identifiers in trace or ledger storage, not uncontrolled metric labels. Aggregate metrics for dashboards while retaining drill-down records with access controls.

## Unit economics and delayed outcomes

Pick a unit that matches the product decision:

- cost per page extracted;
- cost per accepted invoice;
- cost per resolved support case;
- cost per correctly cited answer;
- cost per approved draft;
- cost per avoided manual minute, measured rather than assumed.

Outcomes often arrive later than usage. Join inference attempts to review, correction, payment, or workflow events through stable IDs. Maintain cohorts by model, tenant, document class, and policy version.

Separate:

- **attempt cost** from **completed-unit cost**;
- **completed unit** from **accepted outcome**;
- **acceptance** from **business value**;
- **gross savings hypothesis** from verified realized savings.

A routing change that lowers inference cost but doubles expert review is a regression.

## Budgets are policies, not alerts

Use layered controls:

- **forecast:** expected demand, unit size, price, and uncertainty;
- **soft budget:** notify owners and recommend remediation;
- **hard budget:** enforce admission, downgrade, queue, or stop eligible work;
- **safety reserve:** capacity and spend protected for critical traffic;
- **emergency kill switch:** stops a model, tenant, tool, or agent loop.

Scope budgets by tenant, product, environment, workload, and time window. Enforce them before expensive work starts, using conservative token and tool estimates. Settle actual usage afterward.

Do not apply one blunt cutoff. When a budget is constrained:

1. block non-production experiments;
2. reduce optional sampling or enrichment;
3. route eligible low-risk work to evaluated cheaper models;
4. batch delay-tolerant jobs;
5. cap response length or context through product-aware policies;
6. preserve contractual and safety-critical workloads;
7. abstain honestly when no approved route remains.

Never bypass residency, authorization, quality, or clinical-safety requirements to save money.

## Capacity portfolio

Treat capacity like a portfolio:

| Capacity source | Strength | Risk |
|---|---|---|
| On-demand API | Elastic and low operational burden | Variable price, quota, provider dependency |
| Committed provider capacity | Predictable access and discount potential | Stranded commitment if demand is low |
| Owned or leased accelerators | Control and stable high-volume economics | Capital, operations, utilization, refresh risk |
| Spot/preemptible compute | Low-cost batch capacity | Interruption and uncertain availability |
| Edge inference | Bandwidth, latency, and privacy benefits | Fleet heterogeneity and model constraints |
| Human review | Handles ambiguity and policy exceptions | Scarce expert capacity and queue delay |

Match source to workload class. Interactive work values latency and availability; batch evaluation values price and checkpointability; regulated workloads value eligibility and locality; bursty experiments value elasticity.

Calculate both:

- **marginal cost:** the cost of one additional unit now;
- **fully loaded cost:** direct cost plus allocated commitment and platform overhead.

Routing decisions generally use marginal cost subject to policy and capacity; portfolio decisions need fully loaded cost and utilization.

## Forecasting and commitment decisions

Forecast from workload drivers, not only last month’s bill:

[
	ext{demand} =
	ext{business volume}
	imes
	ext{AI invocation rate}
	imes
	ext{units per invocation}
]

Use distributions for context length, output length, image count, concurrency, retries, and acceptance. Model seasonality and launches. Run low, expected, and high scenarios.

Commit only the stable base load. Keep uncertain peaks elastic. Account for regional fragmentation, maintenance, redundancy, model memory, batch efficiency, and safety headroom. A nominal accelerator count is not usable capacity.

Track commitment coverage, utilization, effective unit price, stranded capacity, rejected demand, queue delay, and the cost of failover.

## Cost controls for agents

Agents multiply spend through planning loops, tools, retries, and sub-agents. Enforce:

- maximum steps, wall time, tokens, tool calls, and parallel branches;
- per-tool and per-model spend estimates;
- loop and duplicate-call detection;
- approval for unusually expensive plans;
- a remaining-budget signal visible to the orchestrator;
- independent server-side enforcement that the model cannot override;
- idempotency and retry budgets;
- a complete ledger of attempted and completed work.

A token limit alone does not control a tool that launches a costly job or human review.

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Detailed per-request ledger | Accurate attribution and debugging | Storage, privacy, schema, and reconciliation complexity |
| Simple blended unit cost | Easy planning | Hides tenant, model, and workload differences |
| Aggressive model downgrading | Lower marginal spend | Quality loss, review cost, and fairness regressions |
| High utilization target | Better commitment economics | Queueing, latency spikes, and no failover headroom |
| Hard tenant budgets | Predictable exposure | Abrupt service loss if degradation is poorly designed |
| Shared capacity pool | Better statistical multiplexing | Noisy neighbors and allocation disputes |
| Chargeback | Strong accountability | Gaming and organizational friction |
| Showback | Visibility without billing | Weaker enforcement |
| Long commitments | Lower predictable unit price | Lock-in and stranded capacity |
| Multi-provider portfolio | Resilience and price options | Evaluation, semantic, and operational complexity |

## Failure modes and defenses

| Failure mode | Defense |
|---|---|
| Bill rises but no team owns it | Mandatory tenant/product/workload attribution and unallocated-cost alarms |
| Provider price changes silently | Versioned price cards, effective dates, invoice reconciliation |
| Retries are counted as successful units | Separate attempt, completion, acceptance, and outcome ledgers |
| Cheaper model increases rework | Quality-adjusted cost with review and correction joins |
| Cache savings are overstated | Record logical demand and physical compute separately |
| Shared GPU costs are split by request count | Allocate using measured resource drivers and batch share |
| Hard cap blocks safety-critical work | Reserved budget and priority-aware degradation |
| Agent loops spend uncontrollably | Step/tool/token/time budgets, loop detection, server kill switch |
| Reserved capacity looks cheap while idle | Report stranded commitment and effective utilized unit price |
| Teams game cost by shortening evaluations | Pair cost goals with quality, safety, and SLO guardrails |
| Finance and telemetry totals disagree | Reconciliation pipeline with explicit rounding, credits, taxes, and lag |
| Cost dashboard leaks tenant prompts | Store billing dimensions without sensitive content; restrict drill-down |
| Forecast misses launch or seasonality | Driver-based scenarios, uncertainty bands, and frequent reforecasting |

## Construction example: invoice and RFI assistant

A construction company processes invoices, RFIs, submittals, and drawing questions. The platform can use a small extraction model, a premium multimodal model, a GPU batch pool, and project-manager review.

The cost ledger attributes every attempt to company, project, workflow, and model. An invoice is not counted as a successful unit until its fields pass validation and the downstream reviewer accepts it. Review minutes and corrections are joined later.

Clear invoices route to the evaluated small model. Difficult handwriting or drawing references may use the multimodal route. Large drawing-set indexing is checkpointed and scheduled in batch capacity. Interactive RFI questions retain a protected latency budget.

At month end, a retry bug begins resubmitting failed tool calls. The per-workflow velocity alert and agent step budget stop the loop. A project reaches its soft budget, so optional summaries stop and delay-tolerant indexing queues; approved invoice processing continues. The system does not route confidential project files to an ineligible cheaper provider.

A healthcare-operations version applies the same controls to scheduling or document workflows, but protected capacity, provider eligibility, minimum quality, audit, and PHI constraints remain hard filters before cost optimization.

## Interview prompt

**Design cost governance for a multi-tenant AI operations platform that processes construction invoices and drawings, answers interactive questions, uses several model providers plus a GPU pool, and includes human review. Control runaway agents without violating quality, residency, or SLOs.**

## Worked answer

1. **Clarify value and constraints.** Define workloads, volumes, latency classes, outcome labels, provider contracts, regions, review capacity, and contractual guarantees.
2. **Create workload classes.** Separate interactive, batch, regulated, experimental, and safety-critical traffic with different eligible routes and degradation policies.
3. **Build a usage ledger.** Record every attempt with tenant, workload, model, units, resource time, route reason, retry lineage, price-card version, and later outcome.
4. **Reconcile cost.** Join provider invoices and infrastructure cost to usage, surface unallocated spend, and retain estimated-versus-final states.
5. **Measure unit economics.** Report marginal and fully loaded cost per completed unit and per accepted outcome, including review and rework.
6. **Enforce policy.** Estimate cost at admission, apply tenant/product/agent budgets, reserve critical capacity, and settle actual usage.
7. **Route safely.** Filter by authorization, residency, capability, quality, and SLO before optimizing cost. Use evaluated fallbacks only.
8. **Plan the portfolio.** Commit stable base demand, keep uncertain peaks elastic, schedule checkpointable batch work on cheaper interruptible capacity, and maintain failover headroom.
9. **Control agents.** Enforce server-side limits for steps, tools, tokens, time, retries, and parallelism; detect loops and expose a kill switch.
10. **Operate with guardrails.** Alert on spend velocity, unit-cost regression, stranded commitment, quality-adjusted cost, queue delay, and attribution gaps. Review policy changes like production releases.

**One-line conclusion:** Optimize AI spend only after policy and quality eligibility, using an auditable usage-to-outcome ledger and a capacity portfolio that protects high-value work from both waste and blunt cost cutting.

## References

- [FinOps Foundation Framework](https://www.finops.org/framework/)
- [OpenCost specification](https://www.opencost.io/docs/specification)
- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [AWS Well-Architected Framework — Cost Optimization](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
- [Google Cloud Architecture Framework — Cost optimization](https://cloud.google.com/architecture/framework/cost-optimization)
- [Microsoft Azure Well-Architected Framework — Cost Optimization](https://learn.microsoft.com/azure/well-architected/cost-optimization/)
