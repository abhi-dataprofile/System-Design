# Decision intelligence, optimization, simulation, and AI-assisted operations research

Prediction estimates what may happen. Operations research chooses what to do under constraints. Decision intelligence connects those choices to evidence, approval, execution, and measured outcomes.

A model that predicts dumpster demand, project delay, patient arrivals, or equipment failure does not by itself produce a safe operating plan. The system must represent objectives, hard constraints, uncertainty, competing stakeholders, and the cost of changing a decision after work has begun.

## Mental model: a decision is a governed optimization over possible actions

Treat the decision service as a versioned pipeline:

```text
observed state + forecasts + policies + candidate actions
  -> validate inputs and authority
  -> construct decision model
  -> solve or simulate alternatives
  -> explain objective and constraint effects
  -> human approval or bounded automatic execution
  -> reconcile actual outcome
  -> learn from forecast and decision error
```

Keep five concepts separate:

1. **State:** what is currently known, including freshness and uncertainty.
2. **Decision variables:** what the system may change.
3. **Constraints:** what is physically, legally, contractually, or operationally allowed.
4. **Objective:** how feasible alternatives are ranked.
5. **Outcome:** what actually happened after execution.

A solver does not discover the business objective. People define it. An optimal answer to the wrong formulation is still wrong.

## When to use it

Use optimization when:

- several actions compete for limited people, equipment, time, rooms, vehicles, or budget;
- choices interact, so independently choosing the best item gives a poor global plan;
- hard feasibility rules must always hold;
- the objective contains measurable tradeoffs;
- planners repeatedly perform the same combinatorial reasoning;
- scenarios must be compared before committing resources.

Use simulation when:

- arrivals, service times, travel, failures, or human behavior are uncertain;
- queues and congestion matter;
- a policy must be evaluated across many plausible futures;
- an analytic formula is too simple for the operational process;
- you need stress tests rather than one point forecast.

Use rules or a priority queue when the problem is small, the policy is stable, and the marginal value of a solver would not justify its operational complexity.

Do not automate a decision when objectives are contested, constraints are not understood, outcomes cannot be observed, or the action carries authority the system has not been granted.

## Frame the decision before choosing an algorithm

Write a decision contract:

| Field | Question |
|---|---|
| Decision owner | Who is accountable for the plan? |
| Cadence | Real time, hourly, daily, or weekly? |
| Horizon | How far ahead is the plan binding? |
| Freeze window | Which near-term actions may no longer move? |
| Decision variables | What can be assigned, sequenced, purchased, or deferred? |
| Hard constraints | What must never be violated? |
| Soft constraints | What may be violated for a measured penalty? |
| Objective terms | Cost, lateness, utilization, safety, fairness, or service level? |
| Uncertainty | Which inputs are forecasts or ranges? |
| Approval boundary | Which changes require a human? |
| Execution boundary | Which system may produce real-world effects? |
| Replanning trigger | Which event makes the plan stale? |
| Outcome | How will value and harm be measured? |

Separate **feasibility** from **preference**. Vehicle capacity, credential requirements, infection-control rules, and locked appointments are hard constraints. Preferred vendors or balanced workloads are usually soft constraints with explicit penalties.

## Decision variables, constraints, and objectives

Suppose binary variable (x_{ij}=1) means resource (i) is assigned to job (j). A simplified assignment model is:

[
min sum_{i,j} c_{ij}x_{ij} + lambda sum_j L_j
]

subject to:

[
sum_i x_{ij}=1 quad 	ext{for each required job } j
]

[
sum_j a_jx_{ij}leq C_i quad 	ext{for each resource } i
]

Here, (c_{ij}) is assignment cost, (L_j) represents lateness, (C_i) is capacity, and (lambda) states how much lateness matters relative to cost.

The formulation must document:

- units and scale of every term;
- why each coefficient exists;
- whether a constraint is hard or penalized;
- who approved the weights;
- how ties are resolved;
- what happens when no feasible plan exists.

Never hide business policy in unexplained weights. A ten-point penalty has no meaning without a unit, baseline, and sensitivity analysis.

## Choosing a solution method

| Method | Good fit | Main limitation |
|---|---|---|
| Greedy heuristic | Fast baseline, simple local choice | Can miss globally better combinations |
| Linear programming | Continuous quantities and linear relationships | Fractional answers may be unusable |
| Mixed-integer programming | Assignment, routing, scheduling, yes/no choices | Worst-case solve time grows quickly |
| Constraint programming | Rich logical and scheduling constraints | Objective modeling and tuning differ from LP/MIP |
| Network flow | Routing, matching, allocation on graph structure | Requires the problem to match flow assumptions |
| Dynamic programming | Sequential decisions with manageable state | State space can explode |
| Metaheuristics | Large hard problems where good-enough is acceptable | No general optimality guarantee |
| Discrete-event simulation | Queues, resources, stochastic process | Evaluates a policy; does not automatically optimize it |
| Simulation optimization | Search policies using simulated outcomes | Expensive and sensitive to simulator fidelity |
| Reinforcement learning | Repeated sequential control with feedback | High data, safety, evaluation, and governance burden |

Start with a transparent baseline. Only add a more complex method when it improves a defined outcome under realistic constraints.

## Forecasts are inputs, not facts

Demand, travel time, service duration, length of stay, and failure risk are uncertain. Preserve their:

- model and feature versions;
- prediction time and horizon;
- confidence interval or quantiles;
- cohort and calibration results;
- data freshness;
- fallback when prediction is missing.

A point estimate can make a brittle plan. Alternatives include:

- safety buffers;
- chance constraints;
- robust optimization over bounded uncertainty sets;
- stochastic programming over weighted scenarios;
- rolling-horizon replanning;
- risk measures such as worst-case loss or conditional value at risk.

Robustness is not free. Excessive buffers waste capacity. Compare the cost of protection with the cost of violation.

## Simulation and policy evaluation

A discrete-event simulation represents events such as job arrival, dispatch, service start, exchange, completion, failure, and resource release. Define:

- entities, resources, queues, and events;
- arrival and service-time distributions;
- routing and priority policy;
- calendars, shifts, travel, breakdowns, and cancellations;
- warm-up period and run length;
- random seeds and number of replications;
- validation against real traces;
- confidence intervals for results.

Use common random numbers when comparing policies so both experience comparable scenarios. Report variation, not only averages. Tail wait time, missed-service probability, overtime, and worst-cohort performance may matter more than mean utilization.

A simulation that matches average volume but not burstiness, correlations, or service-time tails will overstate capacity.

## Architecture

A production decision platform can contain:

1. **State snapshot service** — creates an immutable, time-stamped view of authorized inputs.
2. **Forecast service** — publishes distributions and calibration metadata.
3. **Model builder** — converts the snapshot and policy version into variables, constraints, and objectives.
4. **Solver workers** — run in isolated jobs with time and memory limits.
5. **Scenario simulator** — evaluates candidates across stochastic futures.
6. **Decision registry** — stores input digests, formulation, solver version, parameters, status, and result.
7. **Review surface** — shows changes, binding constraints, alternatives, and uncertainty.
8. **Execution adapter** — applies approved actions idempotently with narrow credentials.
9. **Reconciliation service** — observes actual external outcomes and resolves uncertain writes.
10. **Outcome store** — separates forecast error, planning error, execution error, and environmental change.

A solver result is a proposal until the execution boundary accepts it.

## Feasibility, timeouts, and stale plans

Solvers may return optimal, feasible-but-not-proven-optimal, infeasible, unbounded, timed out, or failed. Preserve the status and optimality gap.

For time-bounded operations:

- keep the best feasible incumbent;
- set a maximum acceptable gap;
- fall back to a deterministic heuristic;
- never treat a timeout as proof of infeasibility;
- extract an infeasibility explanation where supported;
- identify which soft constraints were relaxed;
- require review for emergency relaxation.

Bind every proposal to the exact input snapshot and policy version. Before execution, revalidate locked resources, permissions, capacity, and source revisions. A mathematically valid plan can become unsafe seconds later.

## Explainability for decisions

Useful explanations are operational, not anthropomorphic:

- which constraints are binding?
- why was job A scheduled before job B?
- what would need to change for another choice?
- how much does the objective worsen under the requested override?
- which forecast or policy term drives the choice?
- which alternatives were feasible?
- what uncertainty could reverse the ranking?

For linear models, reduced costs and dual values may help, but they are local mathematical signals, not universal causal explanations. For integer models, use controlled counterfactual re-solves and sensitivity runs.

An LLM may translate solver facts into plain language, but it must not invent constraints, objective values, or causal reasons. Generate the explanation from a structured decision record and verify every number.

## Human overrides and learning

Record an override as a decision:

- original proposal and explanation;
- person, role, and authority;
- changed action;
- reason code and free-text evidence;
- policy exception;
- resulting outcome.

Do not automatically train on overrides as ground truth. A planner may know an unmodeled constraint, prefer a familiar vendor, or simply make a mistake. Review recurring override patterns to improve the formulation.

Distinguish:

- **forecast error:** the estimated demand or duration was wrong;
- **modeling error:** an objective or constraint was missing;
- **solver error:** implementation or numerical failure;
- **execution error:** the approved action was not carried out correctly;
- **environment change:** reality changed after planning.

## Multi-objective decisions and fairness

Cost, speed, safety, emissions, workload balance, and service equity rarely collapse cleanly into one number.

Options include:

- lexicographic priorities: satisfy safety, then service, then cost;
- hard service floors by cohort;
- constrained optimization: minimize cost subject to fairness limits;
- Pareto frontiers showing non-dominated alternatives;
- weighted sums with sensitivity analysis.

Do not use protected or proxy attributes casually. In healthcare operations, fairness may concern access and wait time, but clinical prioritization requires approved policies and clinical governance. In construction, repeatedly delaying small customers or assigning unsafe workloads can be hidden by a globally good average.

## Failure modes

| Failure mode | Defense |
|---|---|
| Optimizes the wrong KPI | Decision contract, outcome review, multi-metric guardrails |
| Feasible mathematically, impossible operationally | Domain constraints, shadow runs, planner review |
| Point forecasts create brittle schedules | Quantiles, scenarios, buffers, rolling replanning |
| Average-valid simulation misses bursts | Validate correlations, tails, and nonstationarity |
| Infeasible model silently drops jobs | Explicit solver status and approved relaxation ladder |
| Timeout is reported as optimal | Preserve gap, incumbent, limit, and status |
| Stale snapshot assigns unavailable resource | Proposal expiry and execution-time revalidation |
| Weight changes alter policy unexpectedly | Versioning, sensitivity analysis, approval |
| One tenant or customer is repeatedly deprioritized | Cohort service constraints and outcome slices |
| LLM invents reasons for a plan | Explanation from structured solver facts |
| Human override becomes noisy training label | Reason capture, adjudication, separate feedback path |
| Replanning causes operational thrashing | Freeze windows, switching penalties, hysteresis |
| Duplicate execution after retry | Idempotency key and outcome reconciliation |
| Simulation is tuned to one historical period | Temporal holdouts and stress scenarios |
| Solver sees unauthorized tenant data | Tenant-scoped snapshots and per-job authorization |
| “Optimal” plan has no fallback | Deterministic safe baseline and manual workflow |

## Tradeoffs

| Choice | Benefit | Cost or risk |
|---|---|---|
| Exact optimization | Quality bound or proof | Longer and unpredictable runtime |
| Heuristic | Fast, simple, resilient | Unknown distance from best plan |
| Frequent replanning | Responds to change | Churn and planner distrust |
| Long freeze window | Stable execution | Slower recovery from new information |
| Robust plan | Fewer violations under uncertainty | Lower nominal utilization |
| Weighted objective | One sortable answer | Policy hidden in coefficients |
| Pareto alternatives | Makes tradeoffs visible | More review burden |
| Human approval | Context and accountability | Latency and inconsistency |
| Automatic execution | Fast response | Requires narrow authority and strong safeguards |
| Detailed simulator | Higher potential fidelity | Calibration and maintenance cost |
| LLM interface | Easier formulation and explanation | Hallucination and prompt-injection risk |

## Construction example: roll-off dumpster allocation and exchanges

A Western New York operator has 20- and 30-yard containers, several trucks, transfer-station hours, driver shifts, weight limits, delivery promises, uncertain pickup readiness, and customers whose containers should not sit idle for weeks.

The daily decision variables assign trucks and containers to deliveries, pickups, and exchanges and sequence each route. Hard constraints enforce container type, truck compatibility, driver hours, disposal-site windows, legal weight, locked appointments, and physical inventory. Soft penalties cover lateness, empty travel, overtime, excessive days at a customer, unnecessary moves, and repeated deferral.

Demand forecasts provide scenarios rather than exact orders. The planner optimizes a stable near-term route with a freeze window and reserves capacity for high-probability exchanges. A discrete-event simulation evaluates the policy under snow, transfer-station queues, heavy loads, delayed customer readiness, truck failure, and demand bursts.

The review screen shows why a pickup moved, which resource is binding, the cost and service effect of an override, and whether the solution is optimal or merely feasible within the time limit. Dispatch approves the plan. Each action carries an idempotency key; the system reconciles the actual driver and disposal outcomes before replanning.

The team measures on-time service, empty miles, container idle days, exchanges per container, tons, overtime, aborted moves, solver runtime, optimality gap, override rate, and performance by customer segment. Revenue is not maximized by blindly preferring heavy loads: safety, legal weight, customer agreements, disposal cost, and long-term service remain explicit constraints and objectives.

## Healthcare operations variation

For operating rooms or infusion chairs, decision variables assign patients, staff, rooms, and equipment over time. Hard constraints cover credentials, cleaning, isolation, equipment, staffing ratios, and approved clinical priority. Duration and cancellation forecasts remain uncertain inputs.

Use simulation to test arrival bursts, overruns, emergency demand, no-shows, room turnover, and staff absence. Optimize operational capacity and access—not diagnosis or clinical treatment. Protect PHI, log access, require clinical governance, preserve emergency/manual workflows, and monitor wait times and cancellations across relevant patient groups.

## Interview prompt

**Design a decision-intelligence platform for a construction company that allocates roll-off dumpsters, trucks, and drivers across deliveries, pickups, and exchanges. Demand, service duration, travel, disposal queues, and weather are uncertain. The system must produce stable plans, support dispatcher overrides, avoid duplicate actions, and explain tradeoffs. Adapt the design to healthcare capacity scheduling.**

## Worked answer

1. **Clarify the decision.** Define owner, cadence, horizon, freeze window, service promises, execution authority, and measurable outcomes.
2. **Model state and uncertainty.** Build an immutable tenant-scoped snapshot of inventory, jobs, trucks, drivers, sites, calendars, weather, and forecast distributions.
3. **Define variables and constraints.** Assign and sequence jobs; enforce compatibility, hours, weight, windows, inventory, and locked work.
4. **Make objectives explicit.** Use lexicographic safety and feasibility, then service, idle days, miles, overtime, and switching penalties with approved units and weights.
5. **Choose a method.** Use MIP or constraint programming for routing/scheduling, a transparent heuristic fallback, and discrete-event simulation for stochastic policy tests.
6. **Handle uncertainty.** Evaluate demand, duration, queue, weather, and failure scenarios; reserve capacity and replan on meaningful events.
7. **Bound computation.** Store solver status, incumbent, gap, and timeout; never call a timed-out feasible solution proven optimal.
8. **Review and explain.** Present binding constraints, feasible alternatives, sensitivity, uncertainty, and structured reasons; capture authorized overrides.
9. **Execute safely.** Revalidate the snapshot, use narrow credentials and idempotency keys, and reconcile outcome-unknown external writes.
10. **Learn carefully.** Separate forecast, formulation, execution, and environment errors; validate changes through replay, simulation, shadow planning, and staged rollout.
11. **Adapt to healthcare.** Add clinical governance, credentials, infection-control constraints, emergency capacity, PHI controls, access metrics, and manual continuity.

**One-line conclusion:** A production decision system does not merely find a low-cost answer—it produces a feasible, uncertainty-aware, explainable, authorized plan whose real-world execution and outcomes can be reconciled.

## References

- [Google OR-Tools documentation](https://developers.google.com/optimization)
- [Pyomo documentation](https://pyomo.readthedocs.io/)
- [SimPy documentation](https://simpy.readthedocs.io/)
- [COIN-OR project](https://www.coin-or.org/)
- [INFORMS — Operations Research and Analytics](https://www.informs.org/Explore/Operations-Research-Analytics)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
