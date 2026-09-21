# RimWorld Advanced Work Planner — Work Planner Design

Colony-level policy, orchestration, route ownership and planning coordination

**Current agreed concept:** v0.44 • 18 September 2026\
**Project:** RimWorld Advanced Work Planner  
**Root namespace:** `RimWorldAdvancedWorkPlanner`  
**Document purpose:** Conceptual specification of colony work-planning policy and orchestration before implementation.  
**Current scope:** Work planning, route ownership, temporal waves, ordinary Primary-first policy, inherited-required recovery, coordinated-wave Primary coverage with context-local planned-Primary state, hierarchical planning/tentative contexts, sequential regret auctions, event-driven route consistency, compression orchestration and route-level optimization coordination.  
**Route algorithm:** Generic route-construction, expansion, compression and cross-route optimization algorithms are specified separately in `RimWorld_Advanced_Work_Planner_RoutePlanner_Design.md`. This document describes only Work Planner policy, invocation conditions, operation inputs/results and higher-level orchestration.  
**Non-goal:** Do not search for a globally optimal colony schedule.

> **Canonical format:** This Markdown file is the source-of-truth design document. Older DOCX files are archival snapshots only.

# 1. Purpose and design philosophy

RimWorld Advanced Work Planner replaces vanilla next-job selection with a lightweight route-based work planner. The system aims to avoid obviously wasteful colony behavior without turning scheduling into an exact global optimization problem.

| **Core principle —** Existing plans are the current baseline, not a different class of route state. A planning session starts as an unchanged child of the real ColonyStateContext and improves complete effective routes using currently unassigned work; redistribution of already planned assigned work remains an explicit transactional operation. Coordinated waves assign important anchors through sequential regret auctions. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

- Walking is the dominant avoidable loss and is represented directly through route time.

- Worker skill and speed affect effective job value, but the user Priority remains the reference continuous-work value rate. A separate global completion-travel bonus, expressed in cells, offsets the inherent bias against short completed jobs.

- Priority expresses value, not urgency. Deadlines and criticality remain a future layer.

- Primary/Backup applies to the complete effective route. With no inherited required work, Work Planner attempts Primary construction/augmentation first and uses Backup as augmentation/fallback according to whether the full route already has Primary. A structural rebuild carrying inherited RequiredItems remains a separate recovery mode: RequiredItems dominate policy, then Primary and Backup are optional augmentation stages.

- Plans are built for temporal waves of pawns that become available close together; isolated pawns are planned independently.

- Normal route generation never steals assigned/reserved work. Cross-route steal/replacement is explicit and transactional and applies only to planned assigned work present in victim routes.

- Shorter plans are acceptable and often desirable: they reduce the amount of future state that can invalidate a plan.

- The generic Route Planner is isolated from RimWorld-specific concepts such as WorkGiver, sleep and Primary/Backup policy.

# 2. Terminology

| **Term**                       | **Meaning**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Work item                      | An abstract planning unit exposed to the Route Planner. It has a start position, fixed or duration-dependent resulting-position functions, pawn-dependent work time and externally supplied Reward. A work item may wrap one raw RimWorld job or a future work package.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| UI Priority / Base Reward Rate | One number entered by the user for each work category/item policy. It is the base continuous-work Reward/s at 100% reference speed and reference result. A separate global CompletionTravelBonusCells setting supplies the short-job completion bias; it is not another per-work Priority.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Primary work                   | Work that provides the pawn local Primary backing and contributes Primary coverage during coordinated multi-pawn planning.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Backup work                    | Work that augments a Primary-backed route and serves as sequential fallback when the complete effective route has no Primary and the Primary construction/augmentation stage cannot obtain one. Backup never numerically competes with a schedulable Primary-stage result.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Forbidden work                 | Work the pawn may not perform.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Anchor                         | A work item used as a required target when generating one candidate route. Anchor is a route-generation constraint, not a reservation type. When an anchor-targeted candidate becomes the selected route, that anchor PlannedStep receives MustRemainAssigned; speculative/lost anchor candidates do not create protection.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Best-local route               | The no-anchor result of WorkPlanner.BuildPolicyRoute. If no route exists, it uses Primary-first construction; if a route exists, it uses Primary-first augmentation of that complete route. Inherited RequiredItems remain the separate structural-recovery mode.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Available job                  | A work item that is currently unassigned and may therefore be considered by normal route construction or augmentation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Assigned / reserved work       | Any WorkItem currently owned rather than unassigned. This includes planned assigned work represented by a PlannedStep and an executing reservation represented outside PlannedRoute. Assigned/reserved work is absent from normal unassigned candidate pools and optimistic free-work matching.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Planned assigned work          | Assigned/reserved work represented by a PlannedStep in a context-effective PlannedRoute. This is the assigned work that route-preserving `TrySteal`/maintenance can redistribute under their documented rules.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Executing reservation          | Assigned/reserved work currently being executed. It is outside PlannedRoute, remains absent from the effective unassigned pool and cannot be a normal candidate or `TrySteal` victim. Event/integration owns the concrete reservation and its completion/release/invalidation lifecycle.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Required job                   | A route-operation constraint. BuildRoute supports the multi-item RequiredJobs recovery/anchor contract; ExpandRoute supports at most one RequiredJob, currently used for inserting one anchor into an existing route. RequiredJobs do not themselves imply MustRemainAssigned.                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| MustRemainAssigned             | Per-PlannedStep preservation decision made by an earlier planning stage. Every selected anchor-targeted route marks its anchor step MustRemainAssigned; speculative/lost anchor candidates and Primary work merely included by best-local do not gain the flag for that reason. Route-preserving maintenance/optimization may move the item atomically but may not drop it while the flag remains valid. On immediate structural rebuild, surviving valid protected work becomes inherited RequiredItems; items that survive the rebuilt result keep the flag, while omitted or invalid/Forbidden items lose that historical protection. The flag is not a colony-global completion obligation.                                                                                                                                                                                                                                                                                                                                                                                                              |
| Planned route                  | The pawn's complete route of future, not-yet-started PlannedSteps in a given context. Executing work is outside this route. Steps have no planning-session provenance; earlier and newly accepted work have identical route semantics.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Temporal wave                  | A group of pawns predicted to become available close enough in absolute time to justify coordinated anchor assignment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Planning context               | A version of the complete colony planning state. ColonyStateContext is the authoritative real state; RootSessionContext starts as its unchanged child. Child contexts expose complete effective routes/ownership plus session changes. Concrete full-state/delta storage and route binding are deferred. |
| Affected configuration         | The pair of routes changed together by one cross-route optimization attempt and accepted atomically if it wins.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Completion travel bonus        | One global/user-configurable distance B measured in map cells. Full WorkItem completion receives extra Reward equal to the pawn effective intrinsic work-value rate multiplied by the travel-equivalent time of approximately B normal traversable cells. It is a completion-bias/tuning parameter, not a literal search radius.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |

# 3. Reward and configuration score

The UI Priority remains the base continuous-work Reward rate; there is no second per-work numerical Priority field. Completion preference is represented by one separate global/user-configurable CompletionTravelBonusCells value B. B is expressed in map cells because its intended meaning is spatial: completing a work item earns extra value roughly equivalent to tolerating that many cells of additional pawn travel. It is not a literal search radius.

P_j = BaseRewardRate_j \[Reward/s\]  
T_j^0 = reference work duration for the represented work  
T_pj = T_j^0 / s_pj  
R_work,pj = P_j \* T_j^0 \* q_pj  
V_intrinsic,pj = P_j \* q_pj \* s_pj \[Reward/s\]  
  
B = CompletionTravelBonusCells \[cells\]  
tau_p(B) = pawn-specific travel-equivalent time for B normal traversable cells  
R_completion,pj = V_intrinsic,pj \* tau_p(B) when the current WorkItem is fully completed  
R_full,pj = R_work,pj + R_completion,pj

For skill-sensitive work, q_pj is a job-family-specific expected-result factor rather than a generic Skill/BestSkill ratio. Quality distributions, botch/fail chance, yield, food poisoning, medical result, resource waste and similar effects require their own expected-outcome models. V_intrinsic,pj is the pawn effective continuous value rate used to convert the spatial completion allowance into Reward. This makes B cells mean the same scheduling tradeoff across pawns with different work speeds/expected results: a completion bonus offsets approximately B cells worth of that pawn's own foregone intrinsic work value. Work families without a meaningful positive planning duration, including instantaneous actions, remain outside the current planner contract pending the critical pre-implementation decision in Sections 20–21.

| **Reference-result baseline —** q = 1 is defined relative to the pawn with the best relevant skill in the colony. Temporary unavailability does not change that baseline. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

For speed-only work, q_pj = 1. tau_p(B) is derived from the pawn movement cost/speed for approximately B normal traversable cells, not from the actual A\* path to the candidate work item; actual detour cost is still represented by ITravelProvider in route elapsed time. For intuition, B = 10 cells roughly compensates an out-and-back detour of about 5 cells from an already useful route, although real route geometry may differ. The Route Planner receives the resulting Reward values from IRewardProvider and returns Q(route) in Reward/s for supported positive-duration routes.

Partial Reward is supplied separately by IRewardProvider using an absolute positive workDuration rather than a semantic completion percentage or a stored fraction of some earlier full-work time. Each supported work family maps that duration for the concrete pawn to whatever durable progress and Reward actually result, which need not be linear. A partial execution that remains incomplete receives no completion-travel bonus. Work Planner does not choose partial planning duration; it consumes the route/result produced by Route Planner.

Colony-level configurations use the same route-rate unit:  
  
Q(configuration) = sum over planned pawns of Q(route_p)  
  
This additive configuration score is also used by two-route steal/replacement. Concrete route and affected-pair ordering is defined by the Route Planner canonical `CompareRoutes` / `CompareRoutePairs` contracts and the Work Planner policy comparators in Section 6.4 rather than restated here. Every non-empty route covered by the current design has `TotalDuration > 0` and uses the ordinary `Q = TotalReward / TotalDuration`; zero-duration route/configuration and regret semantics are outside the current contract pending the critical pre-implementation decision in Sections 20–21.

# 4. Route Planner service boundary

The integration/orchestration layer converts game state into abstract work items and maintains the authoritative planning-state context consumed by Work/Route Planner.

The context hierarchy begins with:

```text
ColonyStateContext
    |
    +-- RootSessionContext        // initially an unchanged child; zero session-local changes
          |
          +-- Work-Planner pawn/candidate operation contexts
                |
                +-- Route-Planner internal branch contexts
```

`ColonyStateContext` represents the current **real planning state of the colony**. It contains/effectively exposes current planned routes, assignments, the unassigned pool, `MustRemainAssigned` flags and all other planning facts materialized into real colony planning state. The future event/integration layer guarantees that this context remains current for every explicitly supported real-game event/change. `PlannedRoute` contains only not-yet-started work; executing work is outside it. These documents specify those state guarantees, not the event-layer operations or storage mechanisms used to maintain them.

A planning session creates `RootSessionContext` as a child of `ColonyStateContext` with no initial changes. Therefore operations in the fresh RootSessionContext see exactly the real colony planning state but cannot change it. Before a normal planning session begins, the event/integration layer guarantees that every persistent route exposed through `ColonyStateContext` has already undergone any required validation/repair and is legal under its authoritative current hard horizon. A route that already exceeds that hard horizon is therefore an integration-contract violation, not an admission/normal-planning branch. v1 assumes the session is atomic with respect to external mutation of ColonyStateContext. Ordinary planning operations may mutate only RootSessionContext descendants/effective alternatives. The **only** operation allowed to change ColonyStateContext from a session is `FinalGlobalCommit`, which atomically materializes the persistent winning session state into the real colony planning state.

A long-lived RoutePlanner instance owns the generic service dependencies used by every route operation:

RoutePlanner  
{  
ITravelProvider  
IRewardProvider  
IAssignmentEligibilityProvider  
}

- ITravelProvider supplies pawn-specific travel duration and reports unreachable endpoint pairs.
- IRewardProvider supplies already-computed worker/work-item Reward for both full and partial execution. The reward model owns base Priority-rate conversion, skill/result modelling and CompletionTravelBonusCells; Route Planner treats it as opaque.
- IAssignmentEligibilityProvider is the policy-aware safety adapter used by generic Route Planner mutations. Work Planner still reads Primary / Backup / Forbidden directly.

PlanningPawnState  
{  
Pawn  
HasPlannedPrimary // cached: true iff the pawn's complete effective PlannedRoute in this context contains >= 1 Primary  
PrimaryCoverageRelaxed // context-local; excludes this unresolved pawn from coordinated coverage matching for the current decision  
}  
  
PlanningContext  
{  
ContextId  
ParentContextId  
PlanningPawns // unresolved entries for active session contexts  
RequiredPrimaryCoverage  
AssignmentDelta // conceptual; concrete full-state/delta encoding is deferred  
CandidateLists  
RequiredLists  
}

Contexts version the **complete colony planning state**. A child context may be implemented internally as a delta, overlay, immutable version or another structure, but semantically every operation can resolve the complete effective route and ownership state for any affected pawn. Individual PlannedSteps do not carry session provenance. Work planned in an earlier session and work added by the current session have the same route semantics once visible in the same effective route.

`HasPlannedPrimary` is only a cache of a property of that complete effective route. Any accepted mutation that can change Primary membership must leave it consistent. `PrimaryCoverageRelaxed` remains a separate Work-Planner decision flag and is never persistent global pawn state.

Shared planning infrastructure owns context lifetime and effective-state resolution. Independently compared Work Planner answers branch from one unchanged common decision baseline. For no-route alternatives that baseline may be the current Work-Planner parent itself; for existing-route alternatives it is normally the shared normalized-baseline child created below that parent. Route Planner may create further descendants below the operation context it receives, releases losing internal branches and merges only its internal winner back into that supplied context. Work Planner likewise destroys losing candidate branches/subtrees before promoting/merging its winner.

Candidate/Required lists are immutable call snapshots addressed by ListId; they bound a planner call but are not authoritative ownership state. Assignment/unassigned-pool state, full effective routes, `HasPlannedPrimary` and relaxation state are context-versioned.

Decision-specific route generation is isolated in child contexts. Work Planner does not run decision-specific `BuildRoute`, `MaximizePartial` or `ExpandRoute` directly against the current Work-Planner parent. A no-route candidate is built in its own child through `BuildRoute`. When compared alternatives share the same existing-route parent baseline, Work Planner first creates one normalized-baseline child, runs `MaximizePartial` there exactly once, and then creates the compared `ExpandRoute` candidate children from that normalized context. The normalized baseline is not promoted into the outer parent merely because normalization succeeded; it reaches the parent only through an accepted winning descendant.

Acceptance is defined over the **entire effective winning path**, not merely the leaf candidate's local delta. If an existing-route candidate wins, the accepted state semantically includes the normalization delta in its normalized-baseline ancestor, the winning candidate delta and any later steal/finalization delta below that path. An implementation must therefore materialize the complete effective winning state when promoting/merging it; it must not merge only the leaf delta and lose an accepted ancestor change. Conversely, if an entire normalized candidate set is abandoned, its whole normalized-baseline subtree is released. Failure of one required-anchor child releases only that child and leaves its shared normalized baseline plus other siblings intact. The concrete merge/delta API remains deferred.

A Work-Planner winning pawn-decision context receives the common immediate steal pass **before it is merged/promoted into its Work-Planner parent for the first time**, provided the pawn has a non-empty effective route. Internal Route Planner merges do not trigger this rule; they may occur repeatedly inside speculative search branches.

Selecting or merging a planning answer is not a real colony-state commit. Within a normal planning session, only `FinalGlobalCommit` materializes the session result into `ColonyStateContext`; event/integration-owned maintenance materialization is intentionally outside this normal-session contract. The exact context representation, full-state/delta encoding, route storage/binding, copy/merge implementation and lifetime strategy are deferred to the dedicated context-layer design.

BuildRoute, MaximizePartial, ExpandRoute, CompressRoute and TrySteal are Route Planner operations. Work Planner decides which operation to invoke and supplies policy-correct candidate/required snapshots and horizon inputs. `BuildRoute` is used when the pawn has no route; `MaximizePartial` explicitly normalizes an existing partial when orchestration requires it; `ExpandRoute` augments an existing route. Planned assigned work redistribution remains the explicit cross-route optimizer; executing reservations are outside Route Planner redistribution.

Route-planner specification: RimWorld_Advanced_Work_Planner_RoutePlanner_Design.md. `WorkPlanner.BuildPolicyRoute` remains a Work-Planner-only policy/orchestration helper; Route Planner does not interpret Primary/Backup.

# 5. Primary / Backup policy

Primary/Backup has two responsibilities. Ordinary planning attempts to establish or preserve Primary backing across the pawn's complete effective route before Backup fallback/augmentation. During coordinated waves, maximum optimistic Primary coverage across remaining unresolved pawns is additionally protected by bipartite matching. A structural rebuild carrying inherited RequiredItems is a separate recovery mode: preserving the maximum achievable inherited-required work takes precedence over establishing Primary backing. Forbidden remains a hard exclusion in every mode. Existing sticky routes are not continuously invalidated merely because new Primary work appears.

## 5.1 Classification matrix

For every relevant pawn/work-item pair, the Work Planner maintains:

Policy\[pawn, workItem\] in { Primary, Backup, Forbidden }  
Capability\[pawn, workItem\] in { can execute, cannot execute }

Forbidden or technically incapable pairs never enter route generation. Primary/Backup classification is read directly by Work Planner and is also available indirectly to Route Planner through IAssignmentEligibilityProvider mutation checks.

## 5.2 Primary Coverage Matching

For an active coordinated PlanningContext, each unresolved `PlanningPawnState` carries context-local `HasPlannedPrimary` and `PrimaryCoverageRelaxed`. `HasPlannedPrimary` is initialized and maintained from the pawn's **complete effective PlannedRoute in that context**. A Primary job counts identically regardless of whether it was planned in an earlier session or the current session.

Relaxed pawns remain unresolved planning participants but are excluded from coordinated Primary-coverage matching for the rest of their current decision. Non-relaxed pawns with `HasPlannedPrimary = true` already satisfy one optimistic Primary-coverage slot and are excluded from the matching side. Matching uses only non-relaxed unresolved pawns that still lack Primary and the effective currently-unassigned work pool visible in the context. v1 intentionally omits path reachability, current horizon, TimeShift requirements and partial-execution support from this graph.

In normal coordinated mode, where no unresolved pawn is relaxed:

```text
covered_context = number of non-relaxed PlanningPawns
                  with HasPlannedPrimary == true
uncovered_context = non-relaxed PlanningPawns
                    with HasPlannedPrimary == false
requiredFromMatching = max(
    0,
    context.RequiredPrimaryCoverage - covered_context)
```

Coverage is protected when:

```text
MaximumPrimaryMatching(
    uncovered_context,
    effectiveUnassignedPool
) >= requiredFromMatching
```

If one or more unresolved pawns are marked `PrimaryCoverageRelaxed`, the fixed target is not used to decide intermediate mutations. Instead every coverage-relevant mutation preserves the current maximum achievable coverage among non-relaxed unresolved pawns:

```text
coverageBefore =
    coveredNonRelaxedBefore
    + MaximumPrimaryMatching(
        uncoveredNonRelaxedBefore,
        effectiveUnassignedPoolBefore)

coverageAfter =
    coveredNonRelaxedAfter
    + MaximumPrimaryMatching(
        uncoveredNonRelaxedAfter,
        effectiveUnassignedPoolAfter)

allow only if coverageAfter >= coverageBefore
```

Matching remains an optimistic coverage oracle rather than a proof of actual schedulability. A pawn stays in PlanningPawns while its candidate and immediate steal pass are evaluated. PrimaryCoverageRelaxed is context-local decision state only.

## 5.3 Ordinary Primary-backed route invariant

Primary/Backup policy is defined over the pawn's **complete context-effective route**. Primary work already present in the route and Primary work added during this session have exactly the same backing/coverage effect.

When Work Planner plans a pawn, it attempts Primary work before Backup work. If the pawn has no route this means Primary-first construction through BuildRoute. If the pawn already has a route this means Primary-first augmentation through ExpandRoute.

A route that already contains Primary is already Primary-backed. Failure to add another Primary does **not** trigger PrimaryCoverageRelaxed; Work Planner may proceed to Backup augmentation. Relaxation is entered only when the pawn's effective route has no Primary and the ordinary Primary stage cannot obtain one for this decision. In coordinated planning, the flag is set before Backup fallback/unchanged-route finalization and remains until that pawn decision is finalized.

Inherited-required recovery remains a separate stronger preservation mode. It may legitimately produce a route without Primary and does not set relaxation merely because optional Primary augmentation fails.

## 5.4 Coverage-preserving matching checks

In normal coordinated mode (`PrimaryCoverageRelaxed == false` for every unresolved pawn), coverage checks are prospective over the complete effective context state after the tested mutation. Let `covered` be the non-relaxed unresolved PlanningPawns whose prospective **whole effective route** contains Primary and `uncovered` the remaining non-relaxed unresolved pawns:

```text
requiredFromMatching = max(
    0,
    context.RequiredPrimaryCoverage - |covered|)

MaximumPrimaryMatching(
    uncovered,
    effectiveUnassignedJobsAfterMutation(context)
) >= requiredFromMatching
```

Multiple Primary items for one pawn satisfy only one slot. Already resolved pawns are absent from PlanningPawns.

When at least one unresolved pawn is relaxed, every coverage-relevant mutation uses the `coverageAfter >= coverageBefore` rule from Section 5.2 over non-relaxed unresolved pawns only. Candidate generation and immediate steal keep the current pawn in PlanningPawns until finalization.

Immediate cross-route steal/replacement does not consume the stolen item from the effective unassigned pool, so CanSteal itself does not run global matching. Its accepted prospective transfer still updates complete effective routes and their HasPlannedPrimary caches. Any work subsequently consumed from the pool during steal passes CanAssign under the current normal/relaxed coverage mode.

When a pawn decision carrying `PrimaryCoverageRelaxed = true` is finalized, Work Planner does not derive the remaining target from the old RequiredPrimaryCoverage. After immediate steal, prospectively remove the finalized pawn and fully recompute:

```text
coveredRemaining = remaining PlanningPawns with HasPlannedPrimary
uncoveredRemaining = remaining PlanningPawns without HasPlannedPrimary

newRequiredPrimaryCoverage =
    |coveredRemaining|
    + MaximumPrimaryMatching(
        uncoveredRemaining,
        effectiveUnassignedPool)
```

For a normal non-relaxed decision, let R be the pre-decision target. If the final effective route contains Primary, the remaining target becomes `max(0, R - 1)`. If it contains no Primary without entering relaxation, prospectively remove the pawn and use `min(R, achievableRemaining)` as before. Finalization is written into the winning child before that child is merged/promoted into its Work-Planner parent.

## 5.5 Singleton and repair planning

Singleton planning and ordinary local repair use the same context machinery but carry `RequiredPrimaryCoverage = 0`, so coordinated matching is vacuously disabled. Primary/Backup policy still applies to the complete effective route. If no route exists, BuildPolicyRoute constructs one; if a route exists, BuildPolicyRoute augments it. Existing Primary anywhere in that route provides normal Primary backing.

## 5.6 Anchor candidate pool

For a coordinated-wave anchor `a`, candidates are drawn only from unresolved pawns currently present in `context.PlanningPawns` for whom `a` is Primary, technically executable, not Forbidden and graph-level coverage-safe. A pawn removed from `PlanningPawns` after its decision is finalized cannot re-enter a later anchor auction in the same planning transaction. Graph-level safety uses the same context-local covered/uncovered model as every other coordinated coverage check; it does not assume that all unresolved pawns still lack planned Primary.

Anchor pools are generated only from a finalized Work-Planner parent decision state. `PrimaryCoverageRelaxed` may exist inside an active pawn-decision candidate branch, but candidate-local relaxation is finalized before merge: the decided pawn is removed from `PlanningPawns` and the remaining coverage target is updated before the next anchor selection begins. Therefore no unresolved relaxed PlanningPawn survives into the parent state from which a coordinated anchor pool is generated, and Section 5.6 correctly uses the normal fixed-target coverage formula rather than the candidate-local relaxation formula.

For each candidate pawn `p`, evaluate the prospective graph state after assigning anchor `a` to `p`. Because `a` is Primary for `p`, prospective `HasPlannedPrimary(p)` is true. Let:

```text
coveredAfter = number of context.PlanningPawns whose prospective
               HasPlannedPrimary is true after p receives a

uncoveredAfter = context.PlanningPawns whose prospective
                 HasPlannedPrimary is false

requiredFromMatching = max(
    0,
    context.RequiredPrimaryCoverage - coveredAfter)
```

Then `p` is graph-level coverage-safe for anchor `a` when:

```text
MaximumPrimaryMatching(
    uncoveredAfter,
    effectiveUnassignedJobsAfterAssigningAnchor(a, p)
) >= requiredFromMatching
```

This naturally handles both cases where `p` was previously uncovered and where `p` already had `HasPlannedPrimary = true`; multiple Primary items for one pawn still satisfy only one slot. The test preserves the actual remaining coverage obligation rather than requiring preservation of the entire current maximum matching. No Backup pawn pool exists for anchors in v1.

There is no passion/skill pruning at the Work Planner layer. If the user classified this work as Primary for a pawn and the pair is technically/coverage valid, that pawn remains in the candidate pool. Skill/result effects may influence route Reward through IRewardProvider, but Work Planner does not reinterpret the user classification.

A work item is graph-level anchor-eligible when it has at least one coverage-safe unresolved `PlanningPawn` Primary candidate. This optimistic matching test does not itself prove that later anchor targeting will succeed. Candidate pawns whose route-appropriate anchor operation — BuildRoute when no route exists, ExpandRoute when one does — returns `Failure` are removed from that auction. If every candidate for every anchor in the current Priority tier fails actual construction, continue to the next lower anchor-eligible Priority tier. A work item that is Backup for every pawn is never a coordinated-wave anchor.

# 6. Route generation modes

All normal Work Planner calls operate against the pawn's complete context-effective route. The route-existence dispatch is strict:

- If the pawn has **no route**, policy construction uses `RoutePlanner.BuildRoute`; no `MaximizePartial` preparation exists because there is no route to normalize.
- If the pawn already has a **non-empty route**, every set of compared alternatives derived from that same parent baseline first gets one dedicated normalized-baseline child context. Work Planner runs `RoutePlanner.MaximizePartial` there exactly once, then creates the compared candidate child contexts and invokes `RoutePlanner.ExpandRoute` in those descendants.
- `MaximizePartial` is tied to establishing a normalized existing-route baseline, **not** to the mere fact that another public `ExpandRoute` call begins. A later public call does not automatically create a new normalization boundary. If an intervening mutation may have exposed additional feasible capacity for an existing partial and Work Planner is about to establish a new normalized planning baseline, normalize that baseline once under the Section 7 rule.

At a Work Planner planning-decision boundary, an absent route and a transiently empty effective route are both treated as **no route** and dispatch to `BuildRoute`. `ExpandRoute` is never the normal decision primitive for an empty baseline. This does not require materializing a synthetic persistent empty route; compression/steal may still produce transient empty snapshots under their Route Planner contracts.

If a no-required `ExpandRoute` finds no augmentation, it still succeeds with its normalized baseline unchanged. If an anchor-required `ExpandRoute` fails, only that anchor candidate child is discarded; the shared normalized-baseline context and sibling alternatives remain valid.

Work Planner stores policy-correct candidate/required snapshots in the operation context and passes ListIds. It consumes Route Planner Success/Failure/result metadata without duplicating Route Planner search internals.

## 6.1 BuildPolicyRoute helper

`WorkPlanner.BuildPolicyRoute(pawn, requiredItems, primaryCandidates, backupCandidates, timeContext)`

BuildPolicyRoute is a Work-Planner-only construct-or-augment helper.

**Ordinary mode — `requiredItems = {}`.**

If the pawn has no route, use the existing Primary-first BuildRoute pipeline: Primary BuildRoute first. When that Primary BuildRoute succeeds, its non-empty result becomes the baseline for one ordinary Backup `ExpandRoute(required = {})`; this following Backup stage has zero positive-shift authority and does not by itself trigger `MaximizePartial`. (Ordinary no-required Primary BuildRoute cannot create a partial, but the no-extra-normalization rule remains the common public-operation boundary rule.) If no Primary route can be constructed, coordinated planning marks relaxation as appropriate and Backup instead becomes the fallback BuildRoute pool. A failed fallback leaves the pawn at ordinary NoRoute/Idle. Therefore Backup augments a successfully constructed Primary-backed route and acts as the sequential from-scratch fallback only when Primary construction fails; these are not competing numeric alternatives.

If the pawn already has a route, BuildPolicyRoute operates from the normalized-baseline child prepared once for this set of alternatives before candidate branching, and first inspects that complete normalized route's `HasPlannedPrimary` state:

- attempt Primary augmentation through `ExpandRoute(required = {})`;
- if the route was already Primary-backed, failure to add another Primary is not relaxation and Backup augmentation may follow normally;
- if the route had no Primary and the Primary stage leaves it without Primary, coordinated planning marks `PrimaryCoverageRelaxed = true` before Backup augmentation/finalization;
- then attempt Backup augmentation through another ordinary ExpandRoute. The mere transition from the Primary stage to the Backup public call is not a new normalization boundary.

Ordinary ExpandRoute does not fail merely because it cannot improve the existing route. The unchanged baseline remains a valid result.

**Inherited-required recovery — `requiredItems != {}`.**

This mode is entered only by structural rebuild after relevant old assignments have already been transactionally released inside the rebuild operation context and surviving protected work has been collected as RequiredItems. Because that rebuild starts from no route, Work Planner calls BuildRoute with the RequiredList and an empty optional candidate list to obtain the best required baseline under Route Planner's multi-required exact-search contract. Surviving protected items represented in the result retain MustRemainAssigned through Work-Planner bookkeeping.

If the required baseline fails, historical protection ends for those unrepresented items and ordinary from-scratch planning continues from the same rebuild context. If it succeeds, a route now exists. A partial created by the required BuildRoute already uses that operation's documented maximum-feasible fallback/shift semantics; merely proceeding to a subsequent public Primary or Backup `ExpandRoute` does not trigger another normalization. Work Planner supplies each later augmentation call from the same fixed normal-session time context plus the then-current effective route; operation metadata may describe required/used shift but does not move the session horizon. If some **intervening mutation** after the required build changes route capacity before a later planning baseline is established, apply the canonical Section 7 normalization-boundary rule at that point. Ordinary optional `ExpandRoute(required = {})` stages receive no positive-shift authority merely because the required build used shift.

## 6.2 Anchor-targeted

Anchor targeting uses one RequiredJob but chooses the Route Planner primitive from route existence:

```text
no existing route:
    candidate child -> BuildRoute(required = {anchor}, Primary candidate pool)

existing route:
    normalized-baseline child -> MaximizePartial once
        -> anchor candidate child
            -> ExpandRoute(required = {anchor}, Primary candidate pool)
```

The selected anchor is Primary for this pawn by the higher-level anchor-candidate rules. For an existing route, ExpandRoute's required list contains exactly this one anchor. Failure means this pawn has no candidate for this anchor: discard only that anchor candidate child. Its normalized-baseline parent and any sibling alternatives remain valid, and no failed anchor mutation reaches the outer Work-Planner parent.

After successful anchor targeting, Work Planner recalculates the correct constraints for any following Backup augmentation from the resulting effective state. The transition to that separate Backup `ExpandRoute` does not itself create another normalization boundary; a partial created by the anchor operation already received that operation's documented partial treatment. Backup is ordinary optional augmentation and therefore receives no positive-shift authority merely because the anchor operation used shift.

When an anchor-targeted candidate becomes the selected pawn decision, Work Planner marks that anchor PlannedStep `MustRemainAssigned = true`.

## 6.3 Best-local

Best-local is ordinary `requiredItems = {}` BuildPolicyRoute. With no route it is the existing Primary-first construction flow. With an existing route it is Primary-first augmentation followed by Backup augmentation as described above. Primary and Backup are policy stages rather than numerically competing classifications.

## 6.4 Canonical comparison policy

Concrete comparison ordering is defined here and in the Route Planner's canonical comparators rather than duplicated throughout orchestration sections.

For comparison only, an `Idle / NoRoute` outside option has the same generic comparison metrics as an empty route: `Q = 0` and `WalkingDuration = 0`. This is a comparison representation only; it does not create a synthetic `PlannedRoute(empty)` and is never persisted into ColonyStateContext.

For alternatives of the **same pawn from the same decision baseline**, Work Planner uses `CompareSamePawnCandidates(A, B)`. Route-bearing alternatives delegate their generic ordering to Route Planner `CompareRoutes`; an Idle/NoRoute alternative participates through the comparison-only metrics above. After that generic comparison:

```text
if generic result != Equivalent:
    return generic result

if exactly one alternative is AnchorTargeted:
    AnchorTargeted wins

return Equivalent
```

Therefore an absolute route tie between an anchor-targeted alternative and its ordinary best-local outside option is deliberately resolved in favor of the anchor-targeted alternative. Algorithm sections reference `CompareSamePawnCandidates` and do not restate Q/walking/tie ordering.

Cross-pawn coordinated-anchor allocation uses a separate `CompareAnchorAuctionCandidates` policy because its primary metric is regret rather than resulting route Q. Section 9.3 defines the score inputs and the comparator applies them in canonical order: higher `Score_p`, then lower `DeltaWalking_p`, otherwise `Equivalent`. Other anchor-auction sections reference this comparator instead of duplicating its ordering.

# 7. Absolute time, sleep horizon, partial normalization and TimeShift

Work Planner owns the higher-level time/horizon policy. For a **normal planning session**, it establishes each relevant pawn's session time context when `RootSessionContext` opens. That session context supplies the absolute route-start `InitialPosition`/`StartTime`, `BaseHorizon`, `HorizonEndPosition` and allowed `MaxTimeShift` used by normal planning for that pawn. These higher-level time-policy inputs are fixed for the lifetime of that session; route mutations may change route duration, route end position, terminal travel and the operation-local `requiredShift`, but they do not move the session's BaseHorizon/target/allowance or route-start origin.

Before each public Route Planner call in the normal session, Work Planner supplies operation inputs derived from that fixed pawn session time context plus the current effective route and operation type. A result that used positive shift does **not** move the BaseHorizon of a later public call. Route Planner internal phases may use their documented call-local limit such as `BaseHorizon + requiredShift`, but that is not a mutation of the session time context. `CompressRoute` used by event/integration-owned maintenance is outside this normal-session invariant and receives the maintenance transaction's currently derived target Horizon.

The exact formulas for choosing the session BaseHorizon, sleep target and MaxTimeShift are intentionally deferred to a dedicated time/horizon design. The current contract fixes their values once a normal session opens. A later normal session may establish different values, and an event/integration-owned maintenance lifecycle may derive its own current preservation constraints. Route Planner does not return or persist a new Horizon.

`MaxTimeShift` is interpreted relative to the fixed BaseHorizon supplied from that session time context. TimeShift remains a bounded **completion allowance**, not generic work budget: BuildRoute may use it under the multi-RequiredJobs rules; `MaximizePartial` may use it only when the additional shift fully completes the existing partial WorkItem; `ExpandRoute` may use it only for full completion of its single RequiredJob/anchor. Positive shift is never introduced merely for additional incomplete progress, Q improvement or ordinary optional work.

All absolute-time feasibility follows the Route Planner Section 5 canonical rule: `StartTime + route.TotalDuration + terminal travel <= Horizon`, with terminal travel required to be reachable. For a NoRoute/transient-empty state, route duration is zero and the endpoint is the fixed session `InitialPosition`.

`MaximizePartial` is never called implicitly by ExpandRoute or TrySteal. Work Planner owns normalization. Normal planning has exactly two session-wide normalization barriers: the global pass at session start and the final global pass after `PlanningDone`. Between them, normalization is tied to establishing a **normalized existing-route baseline**, not to public API-call boundaries. When one or more compared alternatives are derived from the same existing-route parent state, Work Planner creates one child normalized-baseline context, applies `MaximizePartial` there once using the operation-specific constraints calculated for that baseline, and branches the relevant `ExpandRoute` candidates from that normalized state. A later public `ExpandRoute` call does not by itself require another normalization. A newly created partial fallback from BuildRoute/required ExpandRoute already received the enclosing operation's documented maximum-feasible partial treatment. If an intervening mutation (for example steal/removal/repair) may expose additional feasible capacity for an existing partial and Work Planner later needs a new normalized planning baseline, apply `MaximizePartial` once when establishing that baseline. Such a mutation does **not** trigger a global re-normalization barrier.

### Session-wide partial normalization and admission

Immediately after `RootSessionContext` is created, and **before temporal-wave discovery**, Work Planner invokes Route Planner `MaximizePartial` for every pawn that currently has a non-empty effective PlannedRoute. Routes without a partial are no-ops; pawns with no route or a transient empty effective route are not passed to `MaximizePartial`. These accepted normalization results are the first route deltas in RootSessionContext, so every later session calculation — predicted availability, temporal clustering, route Q/walking, HasPlannedPrimary initialization and admission — starts from one globally normalized effective colony state.

Temporal-wave discovery then runs over that normalized RootSessionContext. Pawns selected by the wave rules are only session candidates until admission. **This ordering is deliberate in v1:** a pawn whose normalized route later fails admission because normalization used positive shift may still participate in the seed-window/density calculation and therefore may affect which other pawns enter the temporal wave. This is accepted as a simple expected-rare behavior; do not move admission ahead of clustering unless gameplay testing shows that excluded candidates materially distort wave membership.

A candidate is admitted to `PlanningPawns` only when its normalized complete effective route fits the normal BaseHorizon with **zero positive shift**. For a pawn with no route or a transient empty effective route, admission uses comparison-only empty-route feasibility semantics without materializing a synthetic route: `TotalDuration = 0`, `WalkingDuration = 0`, effective `EndPosition = InitialPosition`, and admission uses the canonical absolute-time check `StartTime + Travel(InitialPosition, HorizonEndPosition) <= BaseHorizon` with that travel required to be reachable. If the initial global MaximizePartial pass required positive shift to finish an existing partial, that route remains legally normalized in RootSessionContext but the pawn is excluded from PlanningPawns for this session. By the normal-session integration precondition, an outright route that already exceeds its authoritative hard horizon has been repaired before `RootSessionContext` is created and is not an admission case.

Therefore the start-of-planning invariant is:

> Every pawn admitted to `PlanningPawns` begins actual planning with a globally partial-normalized complete effective route that fits the normal BaseHorizon without positive shift.

This is an **admission-time classification**, not a lifetime zero-shift invariant for that pawn. Later in the same session its effective route may change, and a local pre-decision `MaximizePartial` may introduce positive shift when doing so fully completes the existing partial and the Work Planner-supplied constraints for that call permit it. Such a pawn does not undergo re-admission merely because that later protected normalization legitimately uses shift.

After normal planning reaches `PlanningDone`, Work Planner performs one final global `MaximizePartial(all pawns with non-empty effective PlannedRoute)` pass before `FinalGlobalCommit`. At that point no further route-selection decision follows, so any newly available legal completion opportunity may be materialized without affecting later planning order or candidate comparison.

# 8. Temporal waves and planning events

## 8.1 Availability / route-finish event

When a normal planning trigger occurs around pawn availability or completion of work, Work Planner opens planning from the current ColonyStateContext guaranteed by the future event/execution integration layer. Before that session opens, required validation/repair has already made every persistent route legal under its authoritative hard horizon. PlannedRoute contains only future, not-yet-started work. Normal session planning therefore starts from valid context-effective routes rather than using admission as a repair mechanism; this document does not mirror the integration-layer operations that maintain ColonyStateContext across the execution boundary.

## 8.2 Seed window

Look forward 2 hours in absolute time and collect every pawn whose predicted current route exhaustion/availability belongs in that interval, including the pawn that triggered the session at t1. Every pawn in this initial 2-hour seed window is included in the temporal wave without density filtering. This stage identifies wave/session candidates only; singleton/coordinated scenario selection happens later, after admission has produced the final `PlanningPawns` set.

For every future pawn in the wave, predicted route exhaustion/availability is derived from the same effective **route-start context** supplied by the event/integration layer. `RouteStartTime` is the predicted time at which the pawn can begin its still-planned route, and `InitialPosition` is the corresponding predicted position; if the pawn is currently executing work outside `PlannedRoute`, these values already reflect the predicted end time/result position of that executing work. For a non-empty planned route, `PredictedAvailability = RouteStartTime + PlannedRoute.TotalDuration`; for NoRoute/transient-empty state, `PredictedAvailability = RouteStartTime`. A pawn already free therefore has `RouteStartTime = now`. No separate FutureRoute concept exists. If the pawn has a route, later planning augments that complete route and may insert work anywhere allowed by ExpandRoute. Route-operation StartTime/InitialPosition come from this route-start context, not from the predicted end of the planned route. The detailed execution/event mechanism behind ColonyStateContext remains outside this design.

## 8.3 Wave expansion

If the initial 2-hour seed contains at least two pawns, sort all included predicted absolute availability times and measure offsets from the first pawn. All seed-window pawns are already members of the wave; density is used only to decide whether the wave should expand beyond that initial window:

0 = d1 \<= d2 \<= ... \<= dn  
  
density_n = d_n / (n - 1) // n \>= 2

After the complete initial seed window has been included, test the earliest not-yet-included pawn beyond the current wave in predicted availability order with:

density\_(n+1) = d\_(n+1) / n

If density\_(n+1) \<= density_n, the next pawn does not increase the current average spacing, so include it and repeat with the next earliest not-yet-included pawn. Stop at the first pawn for which density\_(n+1) \> density_n. That first spacing increase deliberately defines the end of the current temporal wave; v1 does not skip across the gap to search for a denser group farther in the future. v1 intentionally has no independent MaxWaveSpan; practical bounds are expected from pawn availability, sleep and later horizon policy, and a separate span cap is deferred until gameplay testing demonstrates a concrete need.

## 8.4 Session planning-context lifecycle and scenario selection

Every planning session creates `RootSessionContext` as an unchanged child of `ColonyStateContext`, regardless of whether the eventual mode is PlanningDone, singleton or coordinated. Work Planner first performs the Section 7 **global** `MaximizePartial(all pawns with non-empty effective PlannedRoute)` pass. Temporal-wave discovery and expansion then run against that normalized RootSessionContext and produce session candidates.

Wave discovery intentionally precedes admission. Therefore a wave candidate that is later excluded because its normalization required positive shift may already have influenced seed membership or density expansion. This v1 behavior is accepted unless gameplay evidence demonstrates that such excluded candidates materially distort clustering.

Only wave candidates whose normalized complete effective route fits the normal BaseHorizon without positive shift are admitted to `PlanningPawns`. No-route/transient-empty candidates use the Section 7 comparison-only empty feasibility semantics rather than a stored empty route. A candidate whose normalization required positive shift remains represented by its normalized route in RootSessionContext but does not participate in ordinary planning during this session.

Only after admission does Work Planner select the planning scenario:

```text
PlanningPawns.Count == 0  -> PlanningDone
PlanningPawns.Count == 1  -> SingletonPlanning
PlanningPawns.Count >= 2  -> CoordinatedPlanning
```

`PlanningDone` skips anchor/Primary/Backup planning but does **not** discard the session: RootSessionContext may already contain valid global MaximizePartial deltas. It proceeds to the final global MaximizePartial pass and FinalGlobalCommit.

For singleton planning, the one admitted pawn uses the Section 10 singleton policy with `RequiredPrimaryCoverage = 0`.

For coordinated planning, each admitted unresolved entry starts with `PrimaryCoverageRelaxed = false`. `HasPlannedPrimary` is initialized from the pawn's complete normalized effective PlannedRoute in RootSessionContext. A pawn whose route already contains Primary therefore begins covered.

For coordinated planning, initialize the fixed optimistic coverage target as:

```text
coveredRoot = count(PlanningPawns where HasPlannedPrimary == true)
uncoveredRoot = PlanningPawns where HasPlannedPrimary == false

RequiredPrimaryCoverage =
    coveredRoot
    + MaximumPrimaryMatching(
        uncoveredRoot,
        effectiveUnassignedPool)
```

RootSessionContext is the session's current orchestration root. Whenever Work Planner needs independently comparable answers, those alternatives branch as siblings from the same unchanged **common decision baseline**. For a no-route decision this may be the current Work-Planner parent itself; for an existing-route decision it is normally the shared normalized-baseline child produced by one `MaximizePartial`. Route Planner may create descendants only below the supplied operation context.

For a pawn decision, Work Planner lets competing operation contexts finish, compares them, chooses a winner and destroys losing siblings. The winning pawn-decision context remains separate from its Work-Planner parent while immediate cross-route optimization runs. The pawn stays unresolved in PlanningPawns during that pass.

**Immediate-steal boundary.** Before this winning pawn-decision context is merged/promoted into its Work-Planner parent **for the first time**, Work Planner runs the one Section 9.4 sequential `TrySteal` pass for that pawn's complete effective route. Internal context merges wholly inside Route Planner do not trigger this rule. After the pass, Work Planner finalizes coverage in the winning child and only then merges/promotes it into the parent.

After immediate optimization, finalize coverage in one of two modes. In both modes, the winning child/candidate context is finalized **before** it is merged/promoted into its Work-Planner parent.

- **Normal mode (`PrimaryCoverageRelaxed = false`):** let `R` be the pre-decision `RequiredPrimaryCoverage`. If the final complete effective route provides Primary for p, set the child's remaining target to `max(0, R - 1)`. If it provides no Primary without having entered ordinary Primary relaxation — most notably an inherited-required recovery that legitimately remains without Primary — prospectively remove p and compute:

```text
achievableRemaining =
    |coveredRemaining|
    + MaximumPrimaryMatching(
        uncoveredRemaining,
        effectiveUnassignedPool)

newRequiredPrimaryCoverage =
    min(R, achievableRemaining)
```

- **Relaxed mode (`PrimaryCoverageRelaxed = true`):** ignore the old target. Prospectively remove p and fully recompute `coveredRemaining + MaximumPrimaryMatching(uncoveredRemaining, effectiveUnassignedPool)` from the final effective state across all remaining unresolved pawns and all currently free work. Store that value as the winning child's `RequiredPrimaryCoverage`.

In either mode, remove p from the winning child's PlanningPawns, write the final RequiredPrimaryCoverage into that child, then merge/promote the already-finalized child into the Work-Planner parent and continue. `PrimaryCoverageRelaxed` disappears with the removed PlanningPawnState.

When normal planning reaches `PlanningDone`, Work Planner first performs one final global `MaximizePartial(all pawns with non-empty effective PlannedRoute)` pass in RootSessionContext; no-route or transient-empty effective states are no-ops and are not passed to `MaximizePartial`. It then performs `FinalGlobalCommit`. FinalGlobalCommit materializes only persistent colony-planning facts from the winning RootSessionContext into ColonyStateContext: effective routes, assignment/unassigned ownership, MustRemainAssigned and other persistent work/route state. A transient empty effective route materializes as **no persistent planned route**, never as a stored synthetic `PlannedRoute([])`. Session-only orchestration facts — PlanningPawns, RequiredPrimaryCoverage, PrimaryCoverageRelaxed, Candidate/Required ListIds and internal branch metadata — are not materialized into ColonyStateContext. v1 assumes ColonyStateContext cannot change externally during the atomic planning session, so no rebase/version-conflict protocol is required. After commit the session tree is destroyed and materialized colony state must not depend on session-local ContextIds or objects.

# 9. Multi-pawn bootstrap by sequential regret auction

A temporal wave coordinates important anchors without solving one global route-combination problem. The coordinated-anchor set and ordering are **not** frozen at the start of the anchor phase. Whenever Work Planner needs the next anchor, it derives the currently unassigned, anchor-eligible, not-yet-processed WorkItems and their current Priority ordering from the latest effective Work-Planner context after all previously accepted route, steal and coverage-finalization changes. Each concrete WorkItem is considered as a coordinated anchor at most once during this anchor phase; if it was already processed, later state changes do not enqueue it again. Newly available unprocessed work may become eligible, while work assigned incidentally or otherwise removed from the unassigned pool disappears from consideration.

Every route alternative that Work Planner intends to compare is generated as a separate Route Planner operation below one unchanged common Work-Planner decision baseline. For no-route alternatives the candidate operations may be direct sibling children of the current parent. For existing-route alternatives the common decision baseline is the shared normalized-baseline child produced by one `MaximizePartial`; the compared `ExpandRoute` candidates are siblings below that child. Route Planner may branch further internally, but returns its selected internal state merged into that candidate's operation context. Work Planner compares the completed alternatives, releases losing branches/subtrees and keeps the complete winning path isolated until any immediate steal pass is finished.

## 9.1 Mandatory highest-priority anchor

From the latest effective context, inspect currently unassigned, anchor-eligible, not-yet-processed WorkItems by current UI Priority tiers from highest to lower until at least one actual anchor-targeted route can be constructed for some coverage-safe Primary candidate. That becomes the mandatory anchor tier for this bootstrap. Any WorkItem actually considered in this coordinated-anchor selection is marked processed for purposes of the at-most-once anchor-phase rule. v1 intentionally does not optimize which constructible anchor is selected within an exactly equal UI Priority tier; any constructible item in that tier may be chosen. If no constructible anchor-targeted route exists in any anchor-eligible tier, skip the mandatory-anchor phase and proceed to best-local planning.

Once a constructible mandatory anchor is selected, it must be assigned in the wave even when its winner would locally prefer best-local work. Protection is not a mandatory-anchor-specific rule: like every selected anchor-targeted route, the eventual winning route marks its anchor PlannedStep `MustRemainAssigned` under the common selected-anchor rule in Section 9.3/Section 11. The protected anchor may later move atomically under the ordinary MustRemainAssigned/CanSteal rules; the flag does not permanently bind it to its original pawn.

## 9.2 Candidate snapshot for one anchor

For the one currently selected anchor a, take one unchanged current Work-Planner parent context and build the full graph-level coverage-safe Primary pawn pool `P(a)` from unresolved PlanningPawns. Different anchors are processed sequentially by Sections 9.1/9.5; this candidate snapshot compares only the current anchor against each pawn's best-local outside option.

For every pawn p in that pool, derive the anchor-targeted candidate and best-local outside option from the same Work-Planner parent state. If p has no route, create isolated sibling candidate children directly from that parent and use `BuildRoute` for the anchor and ordinary no-route BuildPolicyRoute for best-local. If p has an existing route, first create one normalized-baseline child for p and run `MaximizePartial` there once with the operation-specific constraints Work Planner calculates for that normalization call. Then create the anchor-targeted `ExpandRoute(required={a})` child and the best-local child as siblings from that normalized baseline. Before each descendant Route Planner call, Work Planner supplies the same fixed pawn session time context together with the descendant's current effective route; ordinary best-local augmentation receives no positive-shift authority, while the required-anchor operation may receive positive shift only under its normal protected-completion rule. Failure of the required-anchor child discards only that candidate. The normalized baseline and best-local sibling remain valid.

Primary and any permitted Backup augmentation follow Section 6. There is no passion/skill pre-pruning. If all graph-safe candidates fail actual required anchor construction/augmentation, try another anchor in the same Priority tier where possible.

For each pawn define:

```text
B_p = Q(decisionBaseline_p)                  // normalized route when one exists; comparison-only NoRoute baseline otherwise
A_p = Q(final anchor-targeted route for p)
L_p = Q(best-local route for p)

DeltaA_p = A_p - B_p
```

For an existing route, best-local that cannot improve it simply returns the unchanged baseline, so `L_p = B_p`. For a pawn with no route, failed best-local is the Idle outside option with `L_p = 0`.

Nothing from these candidate routes changes ColonyStateContext. Candidate generation does not remove p from PlanningPawns.

## 9.3 Regret metric and winner

Because different pawns may enter the auction with different already-planned routes, cross-pawn anchor opportunity is measured relative to each pawn's **same decision baseline**. Define `decisionBaseline_p` as the normalized effective route when p has an existing route; when p has no route, use the comparison-only NoRoute baseline from Section 6.4 (`Q = 0`, `WalkingDuration = 0`) without creating a synthetic PlannedRoute.

For pawn p:

```text
B_p = Q(decisionBaseline_p)
A_p = Q(final anchor-targeted route for p)
L_p = Q(best-local route for p)

DeltaA_p = A_p - B_p
DeltaA_best = max(DeltaA_p over surviving candidates)
```

The winner score is the sign-reversed form of the previous regret expression:

```text
Score_p = (DeltaA_p - DeltaA_best) - (L_p - A_p)
```

The second term needs no separate baseline subtraction because `L_p` and `A_p` are alternatives for the same pawn from the same decision baseline.

For the cross-pawn walking component, do **not** use absolute route walking. Define the anchor's walking delta from that same decision baseline:

```text
DeltaWalking_p =
    Walking(anchorRoute_p)
    - Walking(decisionBaseline_p)
```

Choose the winner through the canonical `CompareAnchorAuctionCandidates` comparator from Section 6.4. Same-pawn route/candidate ordering is not restated here.

Choose the winning anchor-targeted candidate context, destroy all competing candidate/outside-option contexts, and keep the winner alive without merging it into the Work-Planner parent yet. When an anchor-targeted candidate becomes the selected route, its anchor PlannedStep receives `MustRemainAssigned = true`. The winning pawn remains in PlanningPawns until the Section 9.4 immediate steal pass and coverage finalization complete.

## 9.4 Immediate cross-route improvement for a winning pawn decision

Before a winning pawn-decision child context is merged/promoted into its Work-Planner parent for the first time, Work Planner runs one sequential immediate-steal pass for that pawn's complete effective route. This applies regardless of whether the winning decision created a route from scratch, augmented an existing route or left an existing non-empty route otherwise unchanged. Route Planner internal branch merges are exempt and never trigger this boundary.

The victim set is every **non-empty** route owned by another pawn whose assignments are authoritative/reserved in the effective parent state and therefore unavailable to the receiver, provided that route contains at least one v1 removal-safe step eligible for consideration. ColonyStateContext routes and accepted ancestor-context routes can be victims; unresolved speculative sibling alternatives, transient empty routes and NoRoute states cannot. Receiver and victim must be different pawns.

For every `TrySteal` call, Work Planner derives only the receiver's fixed preservation horizon from the call-entry effective receiver route rather than carrying an earlier operation's used-shift result:

```text
ReceiverStealHorizon(receiverRoute) = max(
    session BaseHorizon,
    receiverStartTime + receiverRoute.TotalDuration
        + TerminalTravel(receiverRoute, receiverHorizonEndPosition))
```

The formula freezes the boundary already occupied by the current legal receiver route, including a receiver normalized with positive shift, without converting unused MaxTimeShift into generic steal budget. A receiver variant may reuse time saved by its own rearrangement but must fit that frozen boundary. Each later sequential victim call derives a fresh receiver limit from the then-current effective receiver route and does not read an earlier operation's `UsedTimeShift` metadata.

Work Planner supplies no victim horizon. Each victim is already a current legal complete effective route under its owning context's authoritative constraints. Under the fixed-topology minimum-travel contract and removal-safe stationary-WorkItem precondition, removing the stolen step cannot increase the victim's `RequiredElapsed`; the resulting victim route therefore preserves that existing legality without Work Planner or Route Planner reinterpreting its horizon policy.

Work Planner visits eligible victims once in one stable deterministic route/list order. That order carries no policy priority and v1 does not optimize it, but the greedy result may depend on it because every accepted steal changes the effective baseline seen by later victims. For each victim Work Planner calls Route Planner `TrySteal` directly against the still-live winning decision context. TrySteal performs only transfer/replacement of planned assigned work; it does not repair the victim or consume additional unassigned work. If an improvement wins, Route Planner merges it back into the supplied winning context, and the next victim sees that updated effective state. There is no comparison of different victims from a common baseline, no victim-order permutation search and no repeat-to-convergence pass.

Each next victim call must observe all earlier accepted releases/transfers and complete route mutations through the effective context. The concrete full-state/delta/snapshot/overlay mechanism is deferred to the context-layer design.

Immediate steal remains post-selection optimization. Speculative route candidates do not individually simulate future steal opportunities before anchor regret, singleton comparison, best-local comparison or other pre-selection scoring.

Only after the sequential pass finishes does Work Planner finalize the pawn decision's coverage state under Section 8.4 and merge/promote the finalized child into its Work-Planner parent. No post-steal global `MaximizePartial` barrier follows. If the steal freed capacity in another pawn's route, that route remains in the returned effective state until it later becomes an existing-route decision baseline, at which point its candidate set receives the normal one-time local normalization; if no such decision occurs, the final session-wide normalization may use the newly available capacity.

## 9.5 Optional anchors

After the mandatory highest-priority anchor is secured, further coordinated anchors are optional. The list and order are not snapshotted: whenever the phase needs the next optional anchor, derive it afresh from the **latest effective Work-Planner context**, considering only currently unassigned, anchor-eligible WorkItems that have not yet been processed as coordinated anchors, and apply the current UI Priority ordering and current coverage-safe pawn pools. Mark the selected WorkItem processed when it is considered, regardless of whether it is ultimately forced. Generate its current coverage-safe Primary pawn pool and route candidates exactly as above.

For an optional anchor, keep only pawns for whom the anchor-targeted candidate beats that pawn's best-local outside option according to `CompareSamePawnCandidates`. If best-local cannot improve an existing route, its outside option is the successful normalized baseline result; when no route exists and best-local fails, the outside option is Idle with Q=0/walking=0.

If no pawn has an anchor-targeted candidate preferred to its best-local outside option by `CompareSamePawnCandidates`, do not force this optional anchor and continue to the next one. This does not make the WorkItem Forbidden or remove it from the planning universe; it may still be assigned later by ordinary route construction/augmentation. Otherwise use the same baseline-normalized anchor decision score from Section 9.3. The winning child receives the common Section 9.4 pre-merge steal pass, then coverage finalization, then merge/promotion into the current Work-Planner parent.

Because next-anchor selection is recomputed from the latest context, an unprocessed WorkItem disappears automatically if it was incidentally assigned in an earlier accepted route, while an unprocessed WorkItem newly returned to the effective unassigned pool may become anchor-eligible and enter the current ordering. A WorkItem already processed as a coordinated anchor is never reconsidered during the same anchor phase even if a later accepted state change makes it unassigned again. If an anchor-targeted candidate wins, its anchor step receives MustRemainAssigned.

## 9.6 Remaining pawns

After mandatory and selected optional anchors have been processed, plan each pawn remaining in `RootSessionContext.PlanningPawns` in **current predicted availability order**.

This order is not frozen when the temporal wave is formed. Each time Work Planner selects the next remaining pawn, predicted availability is derived from the current context-effective routes after all earlier accepted planning/steal changes. Other unresolved pawns are not globally normalized merely to compute this order. The wave/session membership itself is not expanded or reclustered by those changes.

After a pawn has been selected by that current ordering, prepare its decision alternatives under Section 6. If it has an existing route, create one normalized-baseline child and run `MaximizePartial` there before branching its `ExpandRoute` alternatives; this local normalization does not retroactively rerun the choice of which pawn was next. If it has no route, its candidate children use `BuildRoute` directly. Primary-first policy then proceeds as described in Sections 5–6. If the effective route has no Primary and the ordinary Primary stage cannot obtain one, mark `PrimaryCoverageRelaxed = true` before Backup augmentation/finalization.

The winning pawn-decision context receives the common Section 9.4 immediate steal pass if its effective route is non-empty, even when ordinary augmentation left an existing route unchanged. Then finalize coverage inside the child, remove the pawn from PlanningPawns, and merge/promote the finalized child into the current Work-Planner parent.

A pawn with no route for whom construction fails is Idle. An existing valid route is never converted to Idle merely because augmentation found no improvement.

# 10. Singleton planning

Singleton planning is selected only after the session-start global `MaximizePartial(all pawns with non-empty effective PlannedRoute)` pass, normalized temporal-wave discovery and admission leave exactly one `PlanningPawn`. It uses no coordinated maximum matching (`RequiredPrimaryCoverage = 0`). A wave candidate whose session-start normalization required positive shift was excluded during admission and therefore cannot become the singleton PlanningPawn; its normalization delta may still remain in RootSessionContext for FinalGlobalCommit.

Let p be the only PlanningPawn. Work Planner creates the singleton decision subtree from the current Work-Planner parent. If p has an existing route, create one normalized-baseline child and invoke `MaximizePartial` there exactly once; every singleton alternative is then a child of that normalized baseline. If p has no route, alternatives are ordinary sibling candidate children from the current parent and use `BuildRoute` directly.

Generate the ordinary best-local alternative and independently enumerate every technically eligible currently-unassigned Primary work item as an anchor alternative. For an existing route, each anchor uses `ExpandRoute(required={anchor})`; a required-anchor Failure discards only that anchor child. Best-local uses ordinary no-required augmentation and remains a valid Success even when it adds nothing beyond the normalized baseline. For a no-route pawn, anchor alternatives use `BuildRoute(required={anchor})`, while failed best-local construction represents the Idle outside option.

Compare successful alternatives using the canonical `CompareSamePawnCandidates` policy from Section 6.4. If an anchor-targeted alternative is selected, mark its anchor step `MustRemainAssigned`.

If the selected effective route is non-empty, keep the winning pawn-decision child isolated and run the common immediate sequential steal pass before its first merge/promotion into the Work-Planner parent. Singleton has no coordinated coverage target to update: after the selected route/Idle outcome and any applicable steal pass are complete, remove p from `PlanningPawns`, keep `RequiredPrimaryCoverage = 0`, finalize the decision state and merge/promote it into `RootSessionContext`. The session then reaches `PlanningDone`.

If p had no route and every construction alternative fails, the finalized singleton outcome is ordinary no-route/Idle: no steal pass runs, p is removed from `PlanningPawns`, and the finalized no-route decision proceeds to `PlanningDone`. An existing valid route is never converted to Idle merely because no augmentation improves it.

# 11. Assignment, route protection and normal construction

`ColonyStateContext` is the authoritative real planning-state context. Work assigned in its planned routes is unavailable to normal unassigned candidate pools. Work that has moved from `PlannedRoute` into current execution is likewise still assigned/reserved: execution removes it from the future route but does not return it to the effective unassigned pool, Primary matching or normal candidate snapshots. The event/integration layer owns the concrete executing-reservation state and its eventual completion/release/invalidation. RootSessionContext and descendants inherit the authoritative assignment state and may create alternative effective versions without changing ColonyStateContext until the normal-session FinalGlobalCommit.

- Normal Work-Planner dispatch uses BuildRoute when no route exists and constructs from unassigned candidate/required snapshots. For compared existing-route alternatives, Work Planner creates one normalized-baseline child, runs MaximizePartial once there, and creates ExpandRoute candidate children from that baseline.
- MaximizePartial normalizes the one existing partial without changing route membership/order. ExpandRoute augments that normalized complete existing route, never extends an existing partial, and may insert a single required anchor plus ordinary full candidates according to Route Planner rules.
- Assigned work moves between pawns only through explicit cross-route optimization or higher-level maintenance/rebuild contracts.
- No PlannedStep carries session provenance. A step planned in an earlier session and one added in the current session have identical route semantics.

MustRemainAssigned is stored on PlannedStep and therefore naturally lives in ColonyStateContext together with the route. Child contexts inherit it. Whenever an anchor-targeted candidate becomes the selected pawn decision, its anchor step receives MustRemainAssigned=true. Speculative/lost candidates do not create protection.

During route-preserving optimization or maintenance, a protected step may move atomically but may not be dropped, subject to ordinary transfer eligibility/local Primary backing. Before a structural rebuild, protected work is revalidated; surviving protected work becomes inherited RequiredItems, represented items retain protection, and invalid/omitted items lose it under the established rebuild rules.

An empty PlannedRoute is a transient planning result only. ColonyStateContext does not persist a synthetic empty route for Idle. Executing work is outside PlannedRoute by integration-layer invariant; these documents do not prescribe the transition mechanism.

# 12. Cross-route optimization boundary

Route Planner owns the generic mechanics of moving one already-assigned victim item into a receiver route, optionally releasing one unprotected receiver item, and comparing the complete affected pair. Work Planner supplies the two complete effective routes, the receiver's independently calculated fixed preservation horizon, and the invariant that the victim route is current and legal under its authoritative owning-context constraints.

Cross-route optimization performs **no victim repair or post-steal augmentation**. It does not call ExpandRoute, does not consume additional unassigned work and does not attempt to place a released receiver replacement into the victim. A released receiver item simply becomes unassigned in the tentative branch context. Any later augmentation/normalization is a separate explicit Work-Planner orchestration decision using the public Route Planner primitives.

Every stolen item is replanned as full execution in the receiver. Cross-route optimization has no partial fallback and no MaxTimeShift authority. v1 `TrySteal` may remove only WorkItems satisfying the Route Planner Section 3 removal-safe stationary precondition; under that precondition and the fixed-topology/minimum-travel contract, removing such a victim step cannot increase RequiredElapsed and therefore preserves the current legal victim route without a victim horizon input.

CanSteal validates the atomic transfer/replacement state, including hard ownership/capability rules, MustRemainAssigned constraints and local Primary backing. Because the stolen item was already assigned and absent from the unassigned pool, the transfer itself does not run the ordinary global matching check used by CanAssign. A receiver replacement may release work into the effective unassigned pool; later operations that consume such work pass normal CanAssign coverage checks in the then-current context.

There is no generic configurationValidator or receiverRemovalConstraint callback in v1. Within a normal planning session, context-effective winning transfer state remains tentative until the surrounding `FinalGlobalCommit`; event/integration-owned maintenance materialization is outside this statement. The concrete representation of route deltas/snapshots/overlays remains deferred.

# 13. Planned-route validation and compression

This section specifies only the **planner-facing maintenance behavior** and is also part of the normal-session precondition. If a persistent ColonyStateContext route requires validation/repair before a normal planning event, the event/integration layer arranges for the appropriate maintenance flow to be completed **before** opening a new normal `RootSessionContext`. Normal session planning therefore begins only after persistent routes are legal under their authoritative hard horizons. The concrete maintenance transaction lifecycle — who owns it, which context/commit primitive materializes its result into authoritative `ColonyStateContext`, and whether that mechanism is shared with normal-session `FinalGlobalCommit` — is intentionally deferred to the future event/integration-layer design; this section must not be read as defining a second Work-Planner commit API.

When a planned future route reaches a validation/maintenance point, the integration/event-maintenance layer first ensures that the route snapshot is current for all explicitly handled metric-affecting changes and for its actual route-start context. Work Planner derives the applicable preservation constraints from that effective maintenance state using the deferred time/horizon policy. No persistent Horizon or TimeShift is read from PlannedRoute.

If the unchanged planned route satisfies those preservation constraints, keep it for execution. Any shift/allowance relevant to that decision is operation/context input derived by Work Planner rather than persistent route state and does not automatically become optional-work budget.

If the unchanged current route violates the preservation constraints, Work Planner creates a child compression-operation context from the current maintenance/planning state and invokes `CompressRoute` with the correctly derived `InitialPosition`, `StartTime`, target `Horizon` and `HorizonEndPosition`. CompressRoute receives no candidate list and performs no insertion. Work Planner owns the meaning/calculation of that target Horizon; CompressRoute does not interpret TimeShift, sleep allowance or other higher-level timing policy and returns no horizon/shift metadata.

If compression returns `CannotFitProtectedRoute`, the entire compression operation context and its subtree are discarded; the parent state is unchanged. Work Planner then proceeds to full discard/replan from that unchanged parent state, with surviving protected work handled as inherited `RequiredItems`.

If compression succeeds with a **non-empty** route, Work Planner keeps the successful compression operation context alive. Before treating the repaired route as a new normalized planning baseline, apply `MaximizePartial` once only when the preceding repair/compression mutation may have exposed additional feasible capacity for an existing partial under the freshly derived maintenance constraints. This baseline normalization is not repeated merely because Primary and Backup use separate public `ExpandRoute` calls. Work Planner then creates fresh policy-correct candidate snapshots and calls ExpandRoute first with Primary candidates and then with Backup candidates, deriving the correct time constraints for each call from the resulting effective state. These are ordinary optional augmentation stages and receive no positive-shift authority merely because compression or normalization occurred earlier. After those policy-aware augmentation stages, Work Planner accepts/merges the final repaired state according to the normal planning-context rules.

If compression succeeds with an **empty** route, route-preserving repair has preserved nothing and ends at that point. Keep the successful compression operation context — in which all removed still-valid work is already back in the effective unassigned pool — but do **not** run the normal post-compression Primary/Backup `ExpandRoute` sequence. Instead start ordinary from-scratch `BuildPolicyRoute(requiredItems = {})` from that same effective state; Work Planner derives fresh construction time constraints for the new BuildRoute calls. A successful non-empty replacement forms a winning pawn decision and therefore follows the common Section 9.4 pre-merge steal boundary before acceptance. If that from-scratch planning returns `Failure`, the pawn has no future planned route and is represented by the ordinary no-route/Idle semantics according to the transient-empty-route rule.

# 14. Plan invalidation and repair events

Plan maintenance distinguishes changes that invalidate the whole remaining plan from changes that invalidate only individual future steps. When such maintenance is required for persistent colony state before a normal planning session, the event/integration layer ensures the planner-facing repair flow has been completed and its authoritative state is visible before creating that session's RootSessionContext. How the maintenance result is transactionally materialized into `ColonyStateContext` belongs to the future event/integration-layer design rather than this Work Planner algorithm. By contract `PlannedRoute` already consists only of not-yet-started work; executing work is handled outside these planner documents by the future event/execution integration layer. Route snapshots are expected to be kept current by explicit event-driven maintenance rather than by defensive full reevaluation on every planner call.

## 14.1 Full remaining-plan invalidation

Before discarding the remaining route, collect future MustRemainAssigned steps and revalidate their underlying work items. Drop protection for work that is completed, destroyed, otherwise invalid/impossible, technically no longer executable by this pawn, or currently Forbidden by the user. If the same pawn is immediately replanned, release only still-valid old-route assignments transactionally back into the tentative available pool and call BuildPolicyRoute with the surviving protected work as requiredItems. Hard-invalidated items leave the planning universe and are not returned to the pool. In inherited-required recovery, protected items that survive into the rebuilt result retain MustRemainAssigned; omitted still-valid items lose that historical protection. If none of the inherited RequiredItems can be represented, BuildPolicyRoute applies the Failure fallback defined in Section 6.1 and their historical protection ends before ordinary from-scratch planning continues. Any successful non-empty replacement produced by this structural rebuild follows the common Section 9.4 pre-merge steal boundary before the rebuilt route is accepted.

If no immediate replacement route is built for that pawn, v1 releases the assignments together with the discarded route. It does not maintain a colony-global pending-required queue.

Discard the remaining planned route and return the pawn to planning when the execution context changes so broadly that preserving the route is no longer meaningful. v1 examples include:

- the pawn becomes unavailable for normal work, such as incapacitation, mental-state takeover, drafting/forced control or leaving the map;

- the player explicitly overrides the pawn with an incompatible forced action;

- work-policy or schedule changes invalidate the basis on which most of the route was built;

- the sleep horizon changes so strongly that simple bounded compression cannot retain a valid route;

- a large movement/accessibility context change makes the old route ordering no longer a useful baseline.

## 14.2 Local step invalidation

When only one or several future work items become invalid, distinguish an ordinary planner release from hard work-item invalidation. If a still-valid assignment is merely removed by the planner, release it back to the effective unassigned pool under ordinary eligibility rules. If the underlying work has been completed elsewhere, destroyed/despawned or otherwise ceased to exist as a valid planning item, remove it from the route, assignment state and planning universe without returning it to the pool; this hard invalidation cannot be blocked by CanUnassign/local Primary-backing preservation. A third family of events is deliberately deferred to the future event/integration layer: the WorkItem remains globally valid, but this particular pawn can no longer legally keep the assignment because of a pawn-specific policy/capability/allowed-area/accessibility change. The event layer must define how that assignment is forcibly released/returned to the available pool and how the affected route is repaired; v1 Route Planner does not add another mutation API for this case. Preserve the relative order of retained steps and attempt local repair for events already represented by the current contracts.

Local removal/invalidation is a structural route mutation, not a reason for a defensive full RecalculateRoute pass. The mutation updates only the affected local cached components: removed step reward/work duration, adjacent travel legs, any new bridging leg, route end position when needed, and aggregate TotalReward/WorkDuration/WalkingDuration/TotalDuration/Score. If a hard-invalidated step carried MustRemainAssigned, the protection ends because the underlying work no longer exists as a valid planning item.

## 14.3 Repair procedure

1. Remove invalid future steps while maintaining affected cached route metrics incrementally. Ordinary planner releases of still-valid work return those items to the tentative unassigned pool; hard-invalidated/nonexistent work is removed from the planning universe and is not returned. Pawn-specific invalid-assignment events for otherwise globally valid work use the future event-layer semantics described in Section 14.2 rather than being forced through v1 CanUnassign or hard invalidation.

2. Retain every still-valid step and every surviving MustRemainAssigned flag.

3. Use the current route-start context already represented by the effective planning state. If removal of leading future steps changes the first retained step, update its TravelFromPrevious from the route origin and adjust aggregates locally rather than reevaluating unaffected steps.

4. If the retained route is empty, transition to the from-scratch BuildPolicyRoute/BuildRoute path, carrying surviving protected work as inherited RequiredItems when applicable. If the retained route is non-empty, **do not discard it merely because it currently lacks Primary**. It remains the baseline for the existing-route BuildPolicyRoute/ExpandRoute path. If the preceding removal/repair mutation may have exposed additional feasible capacity for an existing partial and this retained route is now being established as a normalized planning baseline, apply `MaximizePartial` once under the Section 7 rule before branching further planning alternatives. Primary then Backup policy is applied as normal augmentation policy.

5. Work Planner derives the current preservation constraints from the effective maintenance state. If the retained route satisfies those constraints, keep it. Any subsequent augmentation is a new Route Planner call for which Work Planner independently derives the correct Horizon/MaxTimeShift inputs; ordinary optional augmentation receives no positive-shift authority merely because preservation itself required allowance.

6. If the retained route violates the current preservation constraints, create a child compression-operation context and call CompressRoute with the correctly derived target `Horizon`. If compression fails, discard that child context and fall through to full discard/replan from the unchanged parent, with surviving protected work handled by inherited-required recovery. If compression succeeds with a non-empty route, keep the successful compression context and apply the same Section 7 normalization-boundary rule once if that mutation may have exposed additional feasible capacity before the repaired route becomes the baseline for further Primary/Backup augmentation. Do not normalize again merely because Primary and Backup are separate public ExpandRoute calls. If compression succeeds with an empty route, route-preserving repair is over: start ordinary from-scratch BuildPolicyRoute from that already-released effective state using freshly derived construction constraints.

A non-empty replacement or accepted pawn decision follows the normal Section 9.4 pre-merge TrySteal boundary when that boundary applies. Route-preserving maintenance that never creates a new Work-Planner pawn-decision child does not independently invent an additional steal pass.

Known v1 limitation — inherited-required recovery does not persist route provenance saying that a later-added Primary was optional. If such a recovered route is repaired again before execution finishes, generic local-Primary preservation in IAssignmentEligibilityProvider may reject dropping its last Primary even when keeping only the inherited required work would otherwise be a valid recovery result. This can produce a suboptimal repair/replan choice, but it does not create invalid ownership or a global planner failure. Tracking extra provenance solely for this rare repeated-repair case is deferred unless gameplay testing shows meaningful impact.

# 15. Sticky execution

Execution-layer boundary: this section states only planner-facing guarantees and validation requirements. It does not specify the operations, storage variables or transition sequence by which the future event/execution integration layer moves work into execution or maintains `executing work ∉ PlannedRoute`; that implementation will have its own design.

Committed plans are sticky. Ordinary Primary-backed policy is enforced through the appropriate construct-or-augment path for the complete effective route; inherited-required recovery follows its stronger preservation semantics. Neither newly appeared Primary work nor a newly higher-value job by itself interrupts a previously legal committed route in v1.

Route consistency with external game-state properties is event-driven. The future integration layer guarantees that `ColonyStateContext` represents the current real planning state before a session begins, including the current planned routes and persistent assignment state. Ordinary planner operations work from effective context state and do not run defensive full-route refreshes for hypothetical changes.

When an explicitly handled event changes work duration/completion expectations, maintenance updates PlannedWorkDuration, CompletesWorkItem and any dependent PlannedReward/result-position/route aggregates consistently. Such refresh must not silently turn a previously valid full step into an unsupported or unplanned partial step; if the new conditions make the step or route invalid under its constraints, maintenance repairs or invalidates the route instead.

## 15.1 Per-step execution validation

Before starting every next PlannedStep, execution integration performs a local safety/validity check from the pawn actual current time, current position and current worker state. PlannedWorkDuration and CompletesWorkItem are cached estimates in the current route snapshot. For a currently full step, event-driven maintenance keeps the cached duration aligned with the estimated full work time while the route remains valid. For a currently partial step, PlannedWorkDuration is also the current execution budget; actual execution may finish earlier if the current remaining full-work time becomes shorter than that budget.

The integration obtains the resulting position for the actual planned execution and travel from there to the current sleep target. For a full step, the pawn may start only when full execution plus terminal travel satisfies the currently allowed boundary. For an incomplete partial step, its maintained `PlannedWorkDuration` may be executed only when that planned incomplete budget plus terminal travel fits inside the **base boundary**; positive `MaxTimeShift` is not available merely to execute additional incomplete progress. If the base boundary is insufficient, positive shift may justify execution only when the WorkItem will actually **complete within the already maintained execution budget** — for example because current remaining full-work time is now shorter than that budget — and that completion plus terminal travel fits within `base + MaxTimeShift`, using only the minimum completion allowance required. The execution layer does not gain authority to increase `PlannedWorkDuration` simply because positive shift exists; if a larger budget must be planned, that belongs to explicit maintenance/normalization before execution. This is the runtime form of the same completion-boundary rule used by planner normalization, and it is a one-step safety guard rather than a request to recalculate the whole route.

If the next step fails that check, do not start it. The remaining planned route returns to the maintenance path: apply the relevant explicit event/invalidation update, attempt compression and higher-level repair orchestration where appropriate, or discard/replan. Any completion allowance considered by the safety check is ephemeral; it is not stored in route metadata and cannot be converted into optional-work or incomplete-progress budget.

If a planned partial step executes, Reward/progress/result position are calculated from the actual duration executed, not from a stored completion fraction. A newly appeared high-value or Primary job alone does not force colony-wide reconsideration; such a trigger would need an explicitly designed event and scope.

# 16. Performance and caching strategy

v1 should not maintain a persistent cache of arbitrary checked job combinations. The number of possible subsets/orders is large, while route value can be invalidated by explicitly handled worker, path or work-item changes.

- Prioritize long-lived caching inside the travel provider, where future A\* path computation is expected to dominate cost.

- Cache endpoint-to-endpoint travel data and later room-to-room/path-fragment structure where useful; let the travel provider own topology-aware invalidation.

- Planned routes may cache local composable metrics such as per-step PlannedReward and TravelFromPrevious plus route aggregates. Route Planner structural mutations update only affected local components and aggregates rather than defensively reevaluating unaffected steps.

- A reverse Primary index may support `IAssignmentEligibilityProvider` matching fast paths. Work Planner owns the coverage target/relaxation policy, while the canonical mutation-level coverage checks and the correctness condition for skipping matching — including effects on both matching edges and covered/uncovered membership — are defined in Route Planner Section 3; do not duplicate a weaker fast-path rule here.

- Allow short-lived memoization of route evaluations inside one planning event if profiling shows repeated candidates.

- Do not add persistent arbitrary route-combination caching until profiling demonstrates that lookup plus invalidation is cheaper than reevaluation.

# 17. Deferred systems and non-goals

| **Deferred item**                          | **Current decision**                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|--------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Work-item clustering                       | Future work, especially for harvesting, hauling and cleaning. Must not redesign the upper planner or generic Route Planner API.                                                                                                                                                                                                                                                                                                                            |
| Deadlines / criticality                    | Future urgency layer. May affect horizon construction, candidate activation and whether sleeping/unavailable pawns may be activated.                                                                                                                                                                                                                                                                                                                       |
| Exact global optimum                       | Explicit non-goal. The planner aims for sensible low-waste plans, not the mathematical optimum.                                                                                                                                                                                                                                                                                                                                                            |
| Advanced route search                      | Two-job lookahead, beam search, relocate, swap cleanup and 2-opt are deferred until gameplay testing shows concrete failure modes.                                                                                                                                                                                                                                                                                                                         |
| Exact skill-result formulas                | Defined per work family later. Architecture requires only an expected Reward value.                                                                                                                                                                                                                                                                                                                                                                        |
| General needs scheduler                    | Food, joy and other needs remain outside scope; sleep currently enters through the planning horizon.                                                                                                                                                                                                                                                                                                                                                       |
| WorkItem granularity / completion boundary | Deferred until IWorkItem is concretized. v1 planner/simulator deliberately assumes elementary WorkItems with already-resolved planning positions and simple valid execution semantics. Completion-bonus behavior depends on what counts as one WorkItem/completion; clustering, interaction-cell choice, packages and other RimWorld-specific decomposition must not be pulled into planner-algorithm review before the simple simulator/core is debugged. |
| Dynamic dependencies / unlocks             | Future work. v1 candidate pools are treated as the work currently visible to the planning call; jobs that are dynamically created/unlocked by completing another job are deferred.                                                                                                                                                                                                                                                                         |
| Event-driven route-maintenance mapping     | Future integration work: define the concrete RimWorld events that affect cached route metrics/validity and the incremental update, repair or invalidation action for each. The architecture assumes event-driven consistency but does not attempt to enumerate every game-state event in v1 planner design.                                                                                                                                                |
| Pawn-specific assignment invalidation       | Future event-layer contract for cases where a WorkItem remains globally valid but the current pawn can no longer keep the assignment because of policy, capability, allowed-area/accessibility or similar pawn-specific changes. Define forced release back to the available pool, MustRemainAssigned handling and subsequent repair/replanning there; do not overload v1 CanUnassign or hard work-item invalidation.                                               |

# 18. Core invariants

- `ColonyStateContext` is the authoritative current real colony planning state. A fresh `RootSessionContext` is an unchanged child and initially sees exactly that state. Planner operations inside a normal planning session never mutate ColonyStateContext directly; only FinalGlobalCommit may atomically materialize that session's winning state into it. Event/integration-owned maintenance materialization remains a separately deferred layer.

- Contexts semantically version the complete planning state. Implementation may use deltas/overlays, but operations resolve complete effective routes and ownership. PlannedSteps have no session provenance.

- PlannedRoute contains only not-yet-started work. Executing work is outside it but remains assigned/reserved and absent from the effective unassigned pool, Primary matching and normal candidate snapshots until the event/integration layer explicitly completes, releases or invalidates that ownership. The event/integration layer guarantees ColonyStateContext is current for explicitly supported real-state changes; Work/Route Planner do not mirror that layer's transition algorithm or reservation representation.

- UI Priority is the base continuous-work Reward/s; CompletionTravelBonusCells is the separate completion-bias setting. Priority expresses value, not urgency.

- Every non-empty route supported by the current design has strictly positive TotalDuration and uses `Q(route) = TotalReward / TotalDuration`; Q(empty) = 0 is an explicit comparison-only convention. Zero-duration WorkItems/routes and their configuration/regret semantics are outside the current planner contract. Concrete route/candidate comparison ordering is centralized in the Route Planner `CompareRoutes` contract and Work Planner Section 6.4 rather than restated by individual algorithms.

- Assigned/reserved ownership is broader than PlannedRoute membership. Planned assigned work is represented by PlannedSteps and may participate in route-preserving redistribution; executing reservations are assigned but outside PlannedRoute, outside the effective unassigned pool and unavailable to Primary free-work matching, normal candidate snapshots and TrySteal until the event/integration layer releases/completes/invalidates them.

- `HasPlannedPrimary` is a context-local cache of whether the pawn's complete effective route contains at least one Primary. Primary work has identical backing/coverage semantics regardless of which session created its step.

- Coordinated waves protect optimistic Primary coverage through RequiredPrimaryCoverage plus context-local HasPlannedPrimary/PrimaryCoverageRelaxed. Relaxation occurs only when the full effective route lacks Primary and the ordinary Primary stage cannot obtain one. Relaxed decisions dynamically preserve achievable coverage of other non-relaxed unresolved pawns and fully recompute the remaining target at finalization.

- Every session begins with a global `MaximizePartial(all pawns with non-empty effective PlannedRoute)` pass in RootSessionContext **before temporal-wave discovery**. Wave membership, predicted availability, admission and later candidate scoring therefore start from one globally normalized effective state. A wave candidate is admitted only when its normalized complete route fits the normal BaseHorizon with zero positive shift. A normalization that legitimately uses positive shift remains in RootSessionContext, but that pawn is excluded from PlanningPawns for the session. This zero-shift rule classifies admission only; a later local protected `MaximizePartial` for an already-admitted pawn may use positive shift when the Work Planner-supplied constraints for that later call allow it, without re-admission.

- Normal Work-Planner policy uses BuildRoute when no route exists. Compared alternatives over an existing route share one normalized-baseline child: Work Planner runs `MaximizePartial` there once and creates the relevant ExpandRoute candidate children from that state. ExpandRoute supports no RequiredJob or exactly one RequiredJob/anchor; multi-required exact search remains BuildRoute-only. Session-wide `MaximizePartial` runs only at session start and session end; accepted steal/capacity-release mutations do not introduce another global normalization barrier.

- Partial normalization is an explicit Work-Planner orchestration step through Route Planner `MaximizePartial`; it is not hidden inside ExpandRoute. ExpandRoute then handles an optional single required anchor followed by ordinary full optional augmentation. If the normalized baseline still contains a partial, the required anchor must fit fully or required Expand fails; v1 never creates a second partial. A route always contains at most one partial PlannedStep.

- Normal-session time context is fixed per pawn for the lifetime of `RootSessionContext`: route-start `InitialPosition`/`StartTime`, BaseHorizon, HorizonEndPosition and MaxTimeShift do not change as planning mutates routes. Route mutations change route duration/end position/terminal travel and may therefore change operation-local required shift, but an earlier operation's shift never moves a later public call's BaseHorizon. New sessions and maintenance lifecycles may establish different constraints.

- TimeShift is a bounded completion allowance. BuildRoute uses shift for RequiredJobs; MaximizePartial may use it only to fully complete the existing partial; ExpandRoute may use it only to fully complete its single required anchor. Positive shift is never **introduced** merely for additional incomplete progress or ordinary optional work. Route Planner does not return a new Horizon; operation metadata may report used/required shift without modifying the fixed normal-session time context.

- Ordinary BuildPolicyRoute is Primary-first over the complete route. Existing Primary anywhere in the route already provides Primary backing. Failure to add another Primary to a Primary-backed route does not cause relaxation; Backup augmentation may follow.

- Anchor targeting uses BuildRoute(required={anchor}) when no route exists and ExpandRoute(required={anchor}) when one exists. Selected anchor steps receive MustRemainAssigned.

- Across different pawns with different baseline routes, the anchor-opportunity term in the auction uses `DeltaA_p = Q(anchorRoute_p) - Q(normalizedDecisionBaseline_p)`. Pawn opportunity cost remains `L_p - A_p`. The winner maximizes `(DeltaA_p - DeltaA_best) - (L_p - A_p)`.

- Predicted availability is derived from the effective route-start context rather than from route existence alone: `RouteStartTime + PlannedRoute.TotalDuration` for a non-empty route, and `RouteStartTime` for NoRoute/transient-empty state. If the pawn is currently executing work outside PlannedRoute, the event/integration layer supplies RouteStartTime/InitialPosition that already predict the end of that execution. No separate FutureRoute abstraction exists.

- Temporal-wave membership is not reclustered after accepted route changes. After anchor phases, the next remaining pawn is selected in predicted availability order derived from the **current effective routes at that moment**, without globally normalizing other unresolved pawns first. After that pawn is selected, its existing-route candidate set receives its one local normalized baseline; this does not rerun the selection order.

- Every winning pawn-decision child with a non-empty effective route receives exactly one immediate sequential steal pass before it is merged/promoted into its Work-Planner parent for the first time. Route Planner internal merges are exempt. Steal operates on complete effective routes and is insensitive to session provenance.

- MustRemainAssigned is part of PlannedStep state in ColonyStateContext and is inherited/versioned with the route. It is not a colony-global completion obligation and remains subject to the established movement/rebuild rules.

- Candidate/Required lists are immutable call snapshots; authoritative ownership, complete effective routes, unassigned pool and coverage caches are context-versioned.

- Terminal travel participates in feasibility but is excluded from route Q/WalkingDuration.

- Existing route snapshots used by Route Planner are context-effective state, not raw game state. External synchronization belongs to the event/integration layer via ColonyStateContext.

- Empty routes are transient planning snapshots only. BuildRoute cannot Success(empty); compression/steal may temporarily produce empty route state under their documented contracts. At a Work Planner decision boundary, absent or transiently empty route state dispatches to BuildRoute, never normal ExpandRoute. For admission/feasibility only, NoRoute/transient-empty uses comparison-only zero-duration semantics with `EndPosition = InitialPosition` and terminal reachability from that origin. Idle is represented without a persistent empty route, and FinalGlobalCommit materializes a transient empty effective result as no persistent route rather than `PlannedRoute([])`.

- Context operations inside a normal planning session are transactional. Route Planner branch selection and Work Planner candidate selection merge only within the session tree. `FinalGlobalCommit` is the sole normal-session merge that changes `ColonyStateContext`; the materialization mechanism for event/integration-owned maintenance is intentionally not specified here.

# 19. High-level flow

| **Stage** | **Action** |
|---|---|
| Session root | Event/integration layer maintains current `ColonyStateContext`. Work Planner creates an unchanged child `RootSessionContext`; no planning operation may mutate ColonyStateContext. |
| Global normalization | Before temporal clustering, run `MaximizePartial(all pawns with non-empty effective PlannedRoute)` in RootSessionContext. Initial wave discovery, admission and scenario initialization therefore start from one globally partial-normalized effective colony state. |
| Temporal wave + admission | Determine the temporal wave from normalized predicted availability **before admission**. A candidate later excluded for positive-shift normalization may therefore still affect seed/density clustering; this is an accepted expected-rare v1 behavior. Admit only wave candidates whose normalized complete route fits BaseHorizon with zero positive shift into PlanningPawns; NoRoute/transient-empty uses comparison-only zero-duration feasibility from InitialPosition. |
| Scenario selection | Choose only after admission: `0 -> PlanningDone`, `1 -> SingletonPlanning`, `>=2 -> CoordinatedPlanning`. Coordinated mode then initializes HasPlannedPrimary/RequiredPrimaryCoverage from the admitted complete routes. |
| Primary policy | For each decision, Primary is attempted before Backup over the complete route. Existing Primary already supplies backing. Relax only when the full route lacks Primary and Primary planning cannot obtain one. |
| Route primitive | No existing/non-empty route -> candidate child uses BuildRoute directly; there is no pre-Build normalization baseline. Existing non-empty route -> when Work Planner establishes a normalized decision baseline, create one normalized-baseline child, run MaximizePartial once with the operation-specific constraints, then create compared ExpandRoute candidate children from that baseline. A later public ExpandRoute does not by itself create another normalization boundary; normalize again only when an intervening mutation may have exposed capacity and a new normalized baseline is being established. Every descendant call uses the pawn's fixed normal-session time context together with its current effective route. Ordinary optional descendants receive no positive-shift authority; a required-anchor descendant may receive positive shift only under its normal protected-completion rule. |
| Mandatory/optional anchors | For the one current coordinated anchor, compare each pawn's anchor candidate with its best-local outside option from the same decision baseline; existing-route pairs share one normalized-baseline child. Across pawns use the Section 9.3 regret inputs and canonical anchor-auction comparator. |
| Select + steal | Keep the winning pawn-decision child isolated. Before its first Work-Planner merge/promotion into the parent, run one sequential TrySteal pass on the complete effective route. Route Planner internal merges do not trigger this. Finalize coverage, then merge/promote the child. |
| Remaining pawns | Re-evaluate predicted availability order from current effective routes each time the next remaining pawn is chosen, without globally normalizing all unresolved pawns. After selection, normalize that pawn's existing-route decision baseline once before candidate branching. Membership is unchanged. |
| Maintenance | Existing-route preservation/repair remains event-driven. Compression and rebuild use transactional child contexts. Persistent routes that violate their authoritative preservation constraints are handled by maintenance before normal session planning; admission of otherwise legal routes, including routes that legally use allowed shift, is governed separately by the session admission rules. |
| PlanningDone + final normalization | Once no further pawn-planning decision remains, run a final `MaximizePartial(all pawns with non-empty effective PlannedRoute)` pass. NoRoute/transient-empty states are no-ops and are not passed to MaximizePartial. This may add valid completion progress but is followed by no further candidate selection. |
| Final commit | `FinalGlobalCommit` atomically materializes only persistent winning planning facts from RootSessionContext into ColonyStateContext. Session-only orchestration metadata is discarded. This is the only normal-session operation that changes real colony planning state; event/integration-owned maintenance materialization is deferred separately. |

# 20. Open design questions

- **Critical pre-implementation decision — zero-duration work.** Every non-empty route and every routable WorkItem in the current design has strictly positive work/total duration. Before implementation of the core planner/simulator begins, integration must either forbid/normalize zero-duration work before it reaches Work/Route Planner, or the design must separately define zero-duration assignment, reward/scoring, route/configuration comparison, anchor regret, RequiredJobs, steal and execution/immediate-replanning semantics. Until that decision is made, zero-duration WorkItems are invalid planner input and implementations must not invent local zero-duration behavior.

- Concrete expected-result functions q for each skill-sensitive RimWorld work family.

- Work-item clustering model and mapping of cluster progress back to Reward.

- Deadline / criticality layer and its effect on horizon construction, waking unavailable pawns and future persistent obligations.

- Whether predicted-availability order remains the best simple v1 order for remaining-pawn decisions after gameplay testing.

- Profiling threshold for caching/reusing Primary matching results beyond the v1 reverse-index/context fast paths in IAssignmentEligibilityProvider.

- Performance budget and profiling thresholds for travel-cache size, route memoization and matching recomputation.

- Concrete event-to-maintenance mapping for RimWorld state changes that affect route cached metrics or validity; architecture is event-driven, but the exact event set belongs to integration design.

- Pawn-specific assignment-invalidation events where the WorkItem remains globally valid but the current pawn loses policy/capability/area/access eligibility: define event-layer forced-release semantics, return-to-pool behavior, protection handling and repair trigger.

- Exact Work Planner time/horizon calculation policy: define how a normal session chooses each pawn's fixed route-start StartTime/InitialPosition, BaseHorizon, HorizonEndPosition and MaxTimeShift when RootSessionContext opens, including sleep-boundary policy. Once chosen they remain fixed for that session. Event/integration-owned maintenance separately derives its current preservation/compression constraints. The current design requires correct inputs but intentionally does not prescribe those formulas yet.

- Concrete ColonyStateContext route data structures and incremental local-metric update helpers after the conceptual PlannedStep/PlannedRoute snapshot model is implemented. Concrete ColonyStateContext/PlanningContext storage, full-state-vs-delta representation, route binding, winning-steal victim/receiver mutation representation, accepted-mutation API for maintaining HasPlannedPrimary/PrimaryCoverageRelaxed, copy/merge behavior, mutable-vs-immutable context handles, lifetime management, pooling/reference strategy and ListId registry belong to a separate context-layer design; the current documents specify only required semantics and isolation invariants. Temporary candidate/branch route snapshots are expected and their existence does not require a route map field in the conceptual PlanningContext.

- Reward-model internals, including remaining-value semantics after partial progress, belong to a separate reward design; Work/Route Planner treat IRewardProvider as opaque.

# 21. Settled v1 decisions / review guardrails

The following points are deliberate v1 decisions. Routine design review should not reopen them without new gameplay evidence, an implementation contradiction or a concrete failure mode.

- **Critical scope guardrail — zero-duration routes.** All Work Planner policy, configuration scoring, anchor regret, temporal orchestration, steal coordination and related invariants currently specified in this document apply only to non-empty routes with `TotalDuration > 0` built from strictly positive-duration WorkItems. Routine review must not report missing zero-duration behavior as a contradiction inside that bounded scope or add a local workaround. This is nevertheless a hard blocker before core planner/simulator implementation: Section 20's integration prohibition/normalization versus separately designed zero-duration semantics decision must be resolved first.

- Inherited-required rebuild is recovery and does not require Primary backing. Primary augmentation remains Q-improvement-only and is attempted before Backup.

- Review guardrail for inherited rebuild protection: `RequiredJobs` is a generic Route Planner construction constraint and does not by itself mean `MustRemainAssigned`. When the rare structural-rebuild fallback carries surviving protected steps as inherited RequiredItems, Work Planner is responsible for preserving/re-attaching that historical step protection on the items that survive the rebuilt result. This bookkeeping does not require a new generic Route Planner RequiredJobs/provenance mechanism.

- Do not introduce MustIncludeJobs, EstablishPrimary, a worse required baseline chosen for easier Primary insertion, or a second Primary expansion after BuildRoute over a Primary-only pool.

- Every anchor-targeted candidate that becomes the selected route marks its anchor PlannedStep MustRemainAssigned, whether the selection came from the mandatory coordinated auction, an optional coordinated auction or singleton anchor comparison. Lost/speculative anchor candidates and Primary work merely included by best-local do not gain the flag for that reason. Existing generic MustRemainAssigned movement/rebuild rules then apply without an anchor-specific lifecycle. Executed partial work still does not create a persistent colony-wide obligation.

- There is no passion/skill pruning in anchor candidate selection; user Primary classification is authoritative at the Work Planner policy layer. Singleton planning separately enumerates every technically eligible currently unassigned Primary work item as an anchor alternative; it does not stop after one anchor or only the highest Priority tier.

- Coordinated anchor candidate pools contain only unresolved pawns still present in the current context.PlanningPawns. A pawn finalized and removed from PlanningPawns cannot be selected for another anchor in that same planning transaction.

- v1 intentionally does not optimize among constructible anchors inside an exactly equal UI Priority tier.

- In anchor regret/optional-anchor comparison, a best-local `Failure` is the Idle outside option only when the pawn has no existing route. For an existing route, ordinary ExpandRoute no-improvement returns the unchanged baseline successfully; it is not Failure or Idle.

- v1 intentionally has no independent MaxWaveSpan; add one only if horizon/gameplay testing demonstrates a real need.

- Terminal travel to HorizonEndPosition is a feasibility reserve and is intentionally excluded from route Q/WalkingDuration.

- Detailed BuildRoute/ExpandRoute/CompressRoute search, compression and partial-execution mechanics belong to the Route Planner design and are not mirrored here. Work Planner may rely on operation results and result metadata, and may restate a Route Planner fact only when Work Planner itself branches on that fact as part of policy/orchestration.

- Horizon, TimeShift and MaxTimeShift are not PlannedRoute properties. Normal-session time-context guardrail: when `RootSessionContext` opens, Work Planner fixes each relevant pawn's route-start `InitialPosition`/`StartTime`, BaseHorizon, HorizonEndPosition and MaxTimeShift for that session. Later route mutations and operation-local used/required shift do not move those higher-level inputs. Every normal-session Route Planner call uses that fixed time context plus the current effective route. New normal sessions and event/integration-owned maintenance may establish different current constraints. Ordinary optional work receives no positive-shift authority; a protected completion operation may use shift only under its documented completion-boundary rule.

- Do not add routine RecalculateRoute/EvaluateRoute calls as protection against unspecified future state changes. Existing route snapshots are assumed current; external changes are handled through explicit event-driven maintenance, including cached PlannedWorkDuration/CompletesWorkItem updates when appropriate.

- Route snapshot metrics should be locally composable and incrementally maintained by structural mutations rather than forcing whole-route recomputation after a local edit.

- IRewardProvider is opaque to the planner; partial-progress reward accounting is not a Work/Route Planner concern.

- Absolute-time feasibility guardrail: Work Planner uses the Route Planner Section 5 canonical formula rather than comparing a duration directly with an absolute horizon. A route fits when terminal travel is reachable and `StartTime + route.TotalDuration + terminalTravel <= Horizon`; NoRoute/transient-empty uses zero route duration and the fixed route-start InitialPosition.

- Session-start validity guardrail: before a normal RootSessionContext is created, the event/integration layer has already validated/repaired persistent routes against their authoritative hard horizons. Admission distinguishes ordinary zero-shift participation from legally shifted routes; it is not a recovery mechanism for an already invalid over-hard-horizon route.

- Temporal-wave/admission ordering guardrail: wave seed/density discovery intentionally occurs before admission. A normalized candidate later excluded because its normalization required positive shift may still influence wave membership of other pawns. Treat this as an accepted expected-rare v1 simplification; do not move admission ahead of clustering without gameplay evidence that the effect is materially harmful.

- Session atomicity guardrail: v1 assumes `ColonyStateContext` cannot change externally while a `RootSessionContext` session is active. No optimistic rebase/version-conflict machinery is required. FinalGlobalCommit materializes only persistent colony-planning facts; session-only orchestration state is discarded with the session tree.

- Maintenance-boundary guardrail: Sections 13–14 define planner-facing compression/rebuild/repair behavior and the precondition that authoritative maintenance is complete before a normal session opens. They do **not** define how an event/integration-owned maintenance transaction is committed/materialized into `ColonyStateContext`; that lifecycle is deferred to the event/integration-layer design. Do not invent a second Work-Planner commit API merely to fill that deferred layer.

- PlanningPawns, their context-local HasPlannedPrimary/PrimaryCoverageRelaxed facts and RequiredPrimaryCoverage belong to the current planning context; there is no ColonyStateContext/global relaxed-pawn set or committed-coverage counter. At coordinated-root initialization every pawn is non-relaxed and RequiredPrimaryCoverage is `coveredRoot + MaximumPrimaryMatching(uncoveredRoot, effectiveUnassignedPool)`. Singleton/ordinary local-repair contexts keep their single affected pawn but set RequiredPrimaryCoverage to zero. If the pawn's complete effective route lacks Primary and the ordinary Primary construction/augmentation stage cannot obtain one, mark only that unresolved pawn PrimaryCoverageRelaxed in the candidate context; while it remains unresolved, matching ignores relaxed pawns and coverage-relevant mutations must not reduce the current maximum achievable coverage of the non-relaxed unresolved set. On finalization, the winning child itself receives the final PlanningPawns/RequiredPrimaryCoverage state before merge: relaxed decisions use a fresh full achievable target, while non-relaxed decisions without Primary use `min(oldTarget, achievableRemaining)`. Candidate/required lists remain immutable call snapshots, while effective assignment/unassigned-pool/HasPlannedPrimary/relaxation state is context-versioned.

- `HasPlannedPrimary` is derived from the pawn's complete effective route in the current context. Any Primary anywhere in that route counts equally; do not introduce session provenance or a planning-boundary exception.


- Immediate steal is tied to the Work-Planner pawn-decision acceptance boundary: every winning pawn-decision child with a non-empty effective route receives one sequential steal pass before its first merge/promotion into the Work-Planner parent. Route Planner internal merges are exempt. Do not restrict this to anchor winners or only to newly constructed routes.


- Pawn-specific invalidation of an otherwise globally valid assignment is a deferred event/integration-layer responsibility. Do not reinterpret it as ordinary v1 CanUnassign or hard work-item invalidation in routine planner review.

- Compression follows the same child-context isolation rule as other planning operations. A failed `CompressRoute` does not leak partially accepted removals into its parent; its operation context and subtree are discarded, and any subsequent full rebuild starts from the unchanged parent state. A successful **non-empty** compressed state may then be augmented through separate Work-Planner-ordered Primary and Backup ExpandRoute calls before the operation context is merged; those calls use the time context of the owning lifecycle — the fixed pawn session context inside a normal session, or freshly derived maintenance constraints inside an event/integration-owned maintenance transaction. A successful **empty** compressed state instead ends route-preserving repair and starts ordinary from-scratch BuildPolicyRoute from that already-released effective state.

- Accepted planning-time route changes and steal results remain context-effective tentative state until the owning transaction's materialization boundary. In a normal planning session that is `FinalGlobalCommit`; event/integration-owned maintenance materialization is deferred separately. Do not directly mutate ColonyStateContext or ancestor context state in planner pseudocode; the future context-layer design will define how winning route versions are represented and later materialized.

- Work Planner must not duplicate Route Planner internal algorithms. It documents invocation conditions, policy-correct inputs, operation outcomes and higher-level handling only. It may rely on Route Planner result metadata and restate externally observable facts only when Work Planner itself branches on them; detailed route-search/compression mechanics remain authoritative exclusively in the Route Planner design.


- Immediate-steal victim membership is defined by assignment visibility in the parent/effective state: consider every **non-empty other-pawn** route whose assignments were reserved/unavailable to the selected receiver and that contains at least one v1 removal-safe candidate step. ColonyStateContext and accepted ancestor routes can be victims; unresolved speculative sibling alternatives and transient empty/no-route states are not authoritative victims. `receiverWorker == victimWorker` is invalid. For each call, derive only the receiver's fixed preservation horizon as the greater of BaseHorizon and the receiver route's current absolute required end, without adding unused MaxTimeShift; this freezes an already-authorized receiver baseline without carrying prior used-shift metadata or granting generic shift budget. No victim horizon is supplied: current victim legality plus the removal-safe minimum-travel proof guarantees that deleting a candidate step cannot increase RequiredElapsed.

- Execution-boundary guardrail: never represent currently executing work as a leading/locked step inside PlannedRoute merely to support steal or horizon calculations. Executing work is outside the route but remains assigned/reserved and unavailable to the effective unassigned pool, Primary matching, normal candidate snapshots and `TrySteal`. The event/execution integration layer supplies the already-correct route-start context: when execution precedes the still-planned route, `RouteStartTime`/`InitialPosition` already predict the end time/result position of that executing work. The concrete reservation/transition mechanics are intentionally not mirrored here.

- Immediate steal is post-processing only after a route candidate has already won higher-level selection. Do not include candidate-specific future steal potential in regret, singleton alternative comparison, best-local selection or other speculative candidate scoring unless gameplay evidence justifies that larger combinatorial search.

- Existing-route normalization guardrail: independently compared alternatives derived from the same existing-route parent state share one child normalized-baseline context. Run `MaximizePartial` exactly once in that child using the operation-specific constraints Work Planner calculates for that call, then create the compared `ExpandRoute` candidates as descendants. Each descendant Route Planner call uses the same fixed pawn session time context and its own current effective route. A required-anchor Failure destroys only that anchor child. Do not duplicate the same MaximizePartial in each sibling candidate, and do not promote the normalized baseline to the outer parent unless an accepted winning descendant carries it. Acceptance of such a descendant includes the complete effective winning path, including the normalization ancestor delta; abandoning the whole candidate set releases the whole normalized subtree.

- Normalization-boundary guardrail: `MaximizePartial` is tied to establishing a normalized existing-route baseline, not to crossing a public `ExpandRoute` API boundary. A partial newly created by BuildRoute or required ExpandRoute already received that operation's documented maximum-feasible treatment. Do not normalize again merely because Primary, Backup or another stage uses a separate public ExpandRoute call. Within a normal session the higher-level time context is fixed, so normalize again only when an intervening route mutation may have exposed additional feasible capacity and Work Planner is establishing a new normalized planning baseline. A later session or maintenance transaction may establish a different time context and performs its own applicable normalization/repair flow.

- Anchor-relaxation lifecycle guardrail: coordinated anchor pools are always built from a finalized parent decision state with no unresolved candidate-local `PrimaryCoverageRelaxed` pawn. Relaxation may exist inside a pawn-decision branch, but finalization removes that pawn and refreshes the remaining coverage target before the next anchor selection. Do not add a relaxation-mode branch to Section 5.6 unless this lifecycle itself changes.

- Dynamic-anchor-selection guardrail: coordinated anchor membership and ordering are recomputed from the latest effective context whenever the next anchor is needed. Previously accepted route/steal/finalization changes may add or remove unprocessed anchor candidates or change their pools/order. Each concrete WorkItem is processed as a coordinated anchor at most once in that anchor phase and is not re-enqueued after later state changes.

- Post-steal normalization guardrail: do not reintroduce a global `MaximizePartial(all pawns with non-empty effective PlannedRoute)` barrier after steal or another generic capacity-releasing mutation. If such a mutation may expose additional feasible capacity for a partial, normalize that route when Work Planner later establishes a new existing-route planning baseline (or through an explicit maintenance flow); otherwise the final session-wide normalization remains sufficient.

- Comparison-policy guardrail: concrete Q/walking/tie ordering lives only in Route Planner canonical comparators and Work Planner Section 6.4. Orchestration sections reference the appropriate comparator rather than duplicating its internal ordering. Idle/NoRoute may use Q=0 and WalkingDuration=0 solely as a comparison representation; this never creates or persists a synthetic empty PlannedRoute.

- Known v1 limitation: a repeated repair of an inherited-required recovery route may over-preserve an optionally added last Primary because route provenance is not stored. Treat this as a safe but potentially suboptimal repair case unless gameplay evidence justifies extra provenance state.

- Empty-route guardrail: empty PlannedRoute is a transient planning snapshot, not a normal ColonyStateContext execution plan. `BuildRoute` cannot Success(empty); at a Work Planner decision boundary, absent or transiently empty effective route state uses BuildRoute, while ExpandRoute is reserved for a non-empty existing baseline. `CompressRoute` may Success(empty); TrySteal may leave an empty victim route after moving its final step. Admission may evaluate such a state through comparison-only zero-duration/`EndPosition = InitialPosition` semantics, but Work Planner never creates a persistent synthetic empty route. Final materialization represents transient empty as absence of a planned route.

- Sequential-steal context guardrail: each next victim optimization must observe the effective planning state after all earlier accepted steals in that pass. Each pawn is exposed through one complete effective route. The current design intentionally specifies only that semantic requirement; full-state/delta/snapshot/overlay mechanics remain deferred to context-layer design.

- Sequential-steal order guardrail: Work Planner uses one stable deterministic victim order with no policy priority. Because accepted steals update the baseline seen by later victims, the final greedy result may depend on that order. v1 deliberately does not search victim permutations or repeat the pass to convergence.

- Removal-safe WorkItem guardrail: v1 `CompressRoute`/`TrySteal` may remove only WorkItems satisfying the Route Planner Section 3 removal-safe stationary precondition. Movement-providing, topology-changing or otherwise removal-sensitive WorkItems are deferred and must not be routed through those operations until their semantics are designed.

- v1 planner/simulator uses elementary WorkItems and a route contains at most one partial PlannedStep. Context cleanup follows ownership: losing Route Planner branches and losing Work Planner candidate operation contexts are destroyed/released before their surviving sibling is merged into a mutable parent; the RootSessionContext remains leased for the active transaction and the entire session tree is released only after FinalGlobalCommit has materialized every required winning fact into ColonyStateContext. Do not reintroduce persistent C_committed/current-worker-credit accounting fields: HasPlannedPrimary is only context-local state of unresolved pawns. Temporary route snapshots are an expected planning artifact whose concrete storage/context binding is deliberately deferred; absence of a route map in PlanningContext is not a gap, and pseudocode context association must not be read as a prescribed mutable ContextId API. Likewise, MustRemainAssigned means a protected step may move only if ordinary transfer/local-backing rules allow it; “may move” is conditional, not guaranteed.
