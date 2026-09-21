# Work Planner + Route Planner — Design Review Instructions

Use these instructions to review the following connected design documents as one system:

- `docs/RimWorld_Advanced_Work_Planner_WorkPlanner_Design.md`
- `docs/RimWorld_Advanced_Work_Planner_RoutePlanner_Design.md`

## System boundary

This is a colony-level work planner for RimWorld.

### Work Planner responsibilities

- colony-level policy and orchestration;
- Primary / Backup / Forbidden policy;
- temporal waves;
- coordinated planning of multiple pawns;
- anchor assignment;
- regret auction;
- planning contexts;
- route ownership;
- immediate cross-route steal;
- maintenance/recovery orchestration.

### Route Planner responsibilities

Route Planner is the lower generic layer and does not directly understand Primary/Backup policy. It owns:

- `BuildRoute`;
- `MaximizePartial`;
- `ExpandRoute`;
- `CompressRoute`;
- `TrySteal`;
- route scoring;
- travel abstraction;
- RequiredJobs;
- partial execution;
- generic assignment eligibility.

Treat both documents as parts of one design. Review not only each document individually, but also the contract between them.

## Design philosophy

The planner is **not** intended to find a mathematically global optimal colony schedule. Its goal is to cheaply and predictably reject clearly poor decisions and produce a sufficiently good local plan.

> Do not search for the global optimum. At each stage, reject clearly less efficient or unprofitable alternatives, preserve a good baseline, and apply bounded local improvements.

Do not automatically treat the following as problems:

- greedy search;
- no exhaustive search for arbitrary optional jobs;
- no beam search;
- no global route-permutation search;
- no X+Y lookahead;
- no 2-opt / swap / relocate cleanup;
- sequential rather than globally optimized steal;
- deterministic-victim-order dependence of a greedy result;
- heuristic seed selection;
- no exact global colony optimization.

These are conscious v1 tradeoffs unless the documents contradict themselves or the behavior produces a correctness bug. Exact/exhaustive search is used only in bounded places where the design explicitly justifies it, notably RequiredJobs.

## Settled principles

Do not propose changing these solely because a more optimal algorithm exists. Always defer to the current documents if a later version refines one of these principles.

### Route state

- `PlannedRoute` contains only future, not-yet-started work.
- Currently executing work is not in `PlannedRoute`.
- An executing WorkItem remains assigned/reserved and does not return to the `effectiveUnassignedPool`.
- Planned assignment and executing reservation are distinct ownership states.
- `TrySteal` operates only on planned assigned work.

### Contexts

- `ColonyStateContext` is the authoritative real colony planning state.
- A normal planning session creates `RootSessionContext` as an unchanged child.
- Alternatives live in child contexts.
- Losing branches cannot mutate parent or sibling state.
- Route Planner may merge only its own winner into the supplied operation context.
- Route Planner never mutates real colony state directly.
- A normal session materializes its result through `FinalGlobalCommit`.
- Concrete context storage/delta/overlay implementation is intentionally deferred.

### Route provenance

- Route semantics do not depend on the planning session that created a step.
- There is no separate “old route” and “future route.”
- There is one complete effective `PlannedRoute`.
- Currently executing work is outside that route.

### Primary / Backup

- Primary/Backup is policy, not a numeric reward multiplier.
- Existing Primary anywhere in the complete effective route provides Primary backing.
- Backup does not compete numerically with an available Primary-stage result; it is a separate policy stage.
- `HasPlannedPrimary` is a context-local cache of a complete-effective-route property.
- `PrimaryCoverageRelaxed` is context-local temporary decision state, not a persistent pawn flag.

### Coordinated Primary coverage

- Optimistic Primary coverage is protected through maximum bipartite matching.
- Matching is an optimistic oracle, not proof of actual route schedulability.
- Normal mode uses `RequiredPrimaryCoverage`.
- Relaxation mode excludes the relaxed pawn and protects the current maximum achievable coverage of other unresolved pawns.
- Mutation-level matching semantics are canonically defined through `IAssignmentEligibilityProvider`.

### Route generation

- No route → `BuildRoute`.
- Existing non-empty route → the existing-route baseline may pass through `MaximizePartial`, then `ExpandRoute`.
- `BuildRoute` and `ExpandRoute` share a private ordinary optional-augmentation helper.
- That helper is not a separate public `ExpandRoute` operation.
- A public API boundary does not by itself create a normalization boundary.

### Partial work

- A route contains at most one partial PlannedStep.
- Ordinary optional work is never created partial.
- BuildRoute partial is only fallback for an incomplete RequiredJob.
- ExpandRoute partial is only fallback for a single RequiredJob and only when the baseline has no partial.
- `MaximizePartial` increases an existing partial but never shrinks it.
- Positive TimeShift is completion allowance, not generic extra-work budget.
- Positive shift cannot be used merely to increase incomplete partial progress.

### Time and horizons

- `StartTime`, `Horizon` and `BaseHorizon` are absolute times.
- Work, travel and shift are durations.
- Canonical feasibility uses:

  ```text
  StartTime + route duration + terminal travel <= Horizon
  ```

- A normal planning session has a fixed session time context.
- Horizon policy does not float arbitrarily during one normal session.
- Shift used by one operation does not move the horizon of a later public operation.
- Internal optional augmentation within one Route Planner operation may use only the call-local shift justified by that operation's protected completion stage.

### Travel

- In a fixed topology snapshot, `ITravelProvider` returns minimum traversable elapsed travel between endpoints.
- `CompressRoute` and `TrySteal` rely on stationary/removal-safe assumptions.
- Movement-providing or topology-changing WorkItems are unsupported by those operations in v1 until their semantics are designed separately.

### MustRemainAssigned

- It is a preservation constraint on a specific `PlannedStep`.
- A protected step may move atomically but may not be dropped.
- Protection does not guarantee transferability.
- Transfer must still pass ordinary eligibility and local Primary-backing rules.
- `RequiredJobs` alone does not imply `MustRemainAssigned`.

### Compression

- `CompressRoute` is a simple recovery fallback, not an optimizer.
- It does not add work.
- It first tries to reduce the existing partial budget.
- It then greedily removes legal unprotected steps.
- If one removal plus minimum partial shrink fits, choose a fitting variant.
- Otherwise choose the removal with the greatest strictly positive horizon-time gain.
- There is no DFS/combinational removal search or post-compression exact optimization.

### Steal

- `TrySteal` is a bounded two-route local optimization.
- It operates on one receiver and one victim.
- It performs transfer or receiver replacement.
- It does not repair the victim.
- It does not automatically augment the victim route.
- It does not consume additional unassigned jobs.
- Work Planner invokes sequential steal in deterministic victim order.
- Victim order carries no policy priority, but final greedy output may depend on it.
- Searching victim permutations is intentionally deferred.
- Steal is post-selection optimization; speculative candidates do not simulate future steal potential.

### Optimization metric

- Route quality is Reward / elapsed route time for routes inside the currently supported scoring scope.
- Walking is a real cost.
- Terminal travel to `HorizonEndPosition` is used for feasibility but excluded from route Q.
- CompletionTravelBonus compensates for structural bias against short jobs.
- Priority expresses value, not urgency.
- Deadlines/criticality are deferred.

## Required review passes

Perform several passes rather than one linear reading.

### 1. Internal consistency

Look for:

- direct contradictions;
- two different definitions of one contract;
- stale leftovers from an older design;
- pseudocode that disagrees with prose or invariants;
- invariants the algorithm can actually violate;
- Success/Failure semantics that differ between sections.

### 2. Work Planner ↔ Route Planner contract

Pay particular attention to:

- BuildRoute versus ExpandRoute dispatch;
- normalization boundaries;
- RequiredJobs semantics;
- partial semantics;
- TimeShift;
- empty/NoRoute semantics;
- context mutation and merge semantics;
- ownership and the unassigned pool;
- executing reservations;
- MustRemainAssigned;
- Primary coverage;
- CompressRoute orchestration;
- TrySteal lifecycle.

### 3. Context correctness

Search for scenarios where:

- a speculative branch leaks an assignment into a sibling or parent;
- winner merge loses an ancestor delta;
- assignment/unassigned-pool state diverges from route membership;
- `HasPlannedPrimary` becomes stale;
- released or stolen work receives incorrect ownership;
- sequential `TrySteal` observes stale route/state.

Do not demand a concrete delta/overlay implementation. It is intentionally deferred. Review only the semantic contract.

### 4. Time correctness

Check:

- absolute times versus durations;
- terminal travel;
- BaseHorizon/Horizon/MaxTimeShift;
- `requiredShift`;
- session-fixed horizon assumptions;
- availability prediction;
- executing work → route-start context;
- partial normalization;
- compression;
- admission.

Look for conceptual off-by-one-style errors where an absolute timestamp is accidentally compared with a duration.

### 5. Coverage correctness

Check:

- `RequiredPrimaryCoverage`;
- covered versus uncovered pawns;
- matching before and after mutations;
- relaxed-pawn lifecycle;
- anchor candidate graph safety;
- differences between CanAssign, CanUnassign and CanSteal;
- finalization after steal;
- routes containing multiple Primary jobs.

### 6. Edge cases

Explicitly consider:

- zero-duration work, subject to any current critical scope guardrail;
- zero-reward work;
- unreachable travel;
- empty route;
- a route becoming empty after compression or steal;
- a pawn already executing work but having no PlannedRoute;
- an existing partial;
- a partial becoming full;
- a protected item as final route item;
- a victim losing its final PlannedStep;
- receiver replacement releasing a Primary job;
- multiple RequiredJobs where only a subset fits;
- RequiredJob partial fallback;
- RequiredJobs plus optional augmentation;
- zero `PlanningPawns`;
- singleton planning;
- a pawn already Primary-backed;
- no Primary possible;
- positive shift;
- a route already beyond BaseHorizon;
- all candidates failing.

## Finding threshold

Do not produce a list of theoretical optimizer improvements. Prioritize:

1. correctness bugs;
2. contradictions;
3. undefined semantics that permit two materially different implementations;
4. contract gaps between Work Planner and Route Planner;
5. cases where documented invariants cannot be maintained;
6. accidental leftovers from older design versions.

If behavior is merely deliberate greedy/suboptimal v1 behavior, describe it as a limitation, not a defect.

Every reported problem should include a concrete scenario, preferably in this form:

```text
Initial state:
Pawn A route = ...
Pawn B route = ...
Unassigned = ...
RequiredPrimaryCoverage = ...

Operation:
...

According to section X:
...

According to section Y:
...

Result:
contradiction / stale state / wrong ownership / undefined outcome.
```

## Deferred design boundary

The following are intentionally deferred and are not defects merely because they are deferred:

- concrete C# class/API shapes;
- PlanningContext delta/overlay storage;
- context memory management/pooling;
- event-to-maintenance mapping;
- WorkItem granularity;
- exact reward functions for every RimWorld job family;
- exact sleep/horizon formulas;
- moving/topology-changing WorkItem semantics;
- deadlines/criticality;
- clustering/packages;
- global optimization.

If a deferred detail is required for correctness of an already-described algorithm, it is a real design gap. Explain why the current contract needs it now.

## Settled-review guardrails

Both design documents contain `Settled v1 decisions / review guardrails`. Treat them as normative.

If a guardrail appears wrong:

- do not ignore it;
- provide a concrete counterexample, correctness failure or internal contradiction;
- explain why the existing guardrail does not cover that scenario.

Do not reopen a settled choice solely because a more complex or more optimal algorithm exists.

## Required output format

Group findings by severity.

### Critical

Correctness or contract problems that can create invalid assignment/state or a fundamentally incorrect planner result.

### Significant

Real design gaps or contradictions that can produce materially different implementations or noticeably incorrect behavior.

### Minor / clarification

Unclear wording, a missing guardrail, or redundant/stale text.

For every finding include:

1. **Title**
2. **Concrete sections in both documents**
3. **Concrete scenario**
4. **Why it is a problem**
5. **Patch to fix it**

Also include:

### False positives / things checked but not considered problems

Record suspicious cases that are already correctly covered by the current contract. This prevents later reviews from repeatedly reporting the same nonexistent issue.

End with a short assessment of:

- whether the documents are internally coherent;
- whether blockers remain before core planner/simulator implementation;
- the 3–5 areas most likely to produce implementation bugs.

Do not rewrite the entire design. The goal is to identify real problems in the existing design, not to design a different planner.
