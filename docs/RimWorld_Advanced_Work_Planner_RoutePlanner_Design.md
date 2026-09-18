# RimWorld Advanced Work Planner — Route Planner Design

Generic route construction, travel-cost abstraction and bounded cross-route optimization with policy-eligibility callbacks but no direct Primary/Backup semantics

**Current agreed concept:** v0.44 • 18 September 2026\
**Project:** RimWorld Advanced Work Planner  
**Root namespace:** `RimWorldAdvancedWorkPlanner`  
**Document purpose:** Specification of generic route construction, expansion, compression, travel-cost abstraction and bounded cross-route steal/replacement optimization  
**Inputs:** Abstract elementary work items, worker context, absolute construction/augmentation times and horizons, unassigned candidate pools, multi-item BuildRoute RequiredJobs, optional single-item ExpandRoute RequiredJob, complete context-effective route baselines, and an infrastructure-owned planning-context reference for each candidate route; policy mutation checks come from the long-lived eligibility provider and effective hierarchical context state.  
**Explicit exclusions:** Direct Primary/Backup interpretation, anchor candidate selection, temporal waves, regret-auction policy and atomic game-state ownership updates. Route search receives only mutation-validity answers through `IAssignmentEligibilityProvider`.  
**Optimization style:** Greedy ordinary optional construction/augmentation, exhaustive multi-RequiredJobs search only for BuildRoute, exhaustive insertion-position handling for one ExpandRoute RequiredJob, completion-boundary TimeShift, incremental local snapshot-metric updates, plus simple greedy removal compression; deliberately not a global exact route solver.

> **Canonical format:** This Markdown file is the source-of-truth design document. Older DOCX files are archival snapshots only.

# 1. Scope and design goal

The Route Planner is a long-lived generic service with five core responsibilities: build one route from a context-owned candidate snapshot, maximize the one existing partial step of a route when policy requests that normalization, expand an existing route from another context-owned snapshot, compress an existing route inside a fixed hard horizon, and perform bounded cross-route steal/replacement between two assigned routes. It does not interpret Primary/Backup policy; every tentative assignment mutation is validated through IAssignmentEligibilityProvider.

- No Primary / Backup / Forbidden enum or matching algorithm is implemented directly inside this component; higher-level policy enters only through context-owned CandidateListId/RequiredListId snapshots and IAssignmentEligibilityProvider yes/no mutation checks.

- Construction requests may use absolute StartTime/Horizon values, while travel/work/shift values are durations. Persistent route structure does not need cumulative absolute per-step timestamps.

- Route quality is Reward/s. IRewardProvider supplies already-computed full/partial Reward; Route Planner does not derive UI Priority, skill/result formulas or CompletionTravelBonusCells.

- BuildRoute receives an empty PlannedRoute carrying ContextId and constructs it from context-owned lists. Required jobs are searched exhaustively over policy-valid ordered subsets/permutations and supported terminal-partial alternatives; ordinary optional work remains greedy. `MaximizePartial` is a separate public normalization primitive for the one existing partial step. ExpandRoute augments a normalized existing route, optionally requiring one anchor, and otherwise greedily adds ordinary full work. CompressRoute is a deliberately simple fallback that preserves/shrinks an existing partial when possible and otherwise removes unprotected work greedily until the route fits; it never adds work. Cross-route optimization is tentative and transactional and performs no victim repair.

- `ColonyStateContext` is the authoritative current real planning state maintained by the event/integration layer. Route Planner receives effective route state through the supplied planning context and assumes the derived cached metrics/operation inputs are current; it does not defensively synchronize arbitrary external game-state changes.

- Structural mutations performed by Route Planner keep affected local cached metrics consistent as part of the mutation itself. A local removal/insertion/replacement is not a reason to reevaluate unaffected steps.

- Ordinary optional construction remains intentionally simple. RequiredJobs are the bounded exhaustive exception; single-route compression is intentionally a simple recovery fallback rather than another exact/local-combination optimizer.

# 2. Work-item and planned-step contracts

A work item is an abstract elementary operation. The minimum full-execution contract is:

IWorkItem  
{  
StartPosition  
EndPosition()  
EndPosition(worker, workDuration)  
GetWorkTime(worker) // signed duration  
SupportsPartialExecution  
}

StartPosition is where execution begins. EndPosition() is valid only for a fixed resulting position; duration-dependent work uses EndPosition(worker, workDuration). GetWorkTime(worker) is full elapsed execution duration after arrival; negative means this worker cannot execute the item, zero is valid for instantaneous work. SupportsPartialExecution is true only when the integration layer can persist meaningful partial progress and provide correct partial Reward/result-position semantics.

Every IWorkItem has stable identity for the lifetime of its planning/assignment existence. Equality/hashing used by assignment ownership, route membership, matching indices, search-state bookkeeping and caches use that identity rather than mutable progress or route metadata.

A `PlannedRoute` contains only future, not-yet-started work. A WorkItem that the pawn is already executing is outside `PlannedRoute` and is never represented by a leading executing step or other locked route prefix. Entering execution does **not** release that WorkItem: while execution owns it, the item remains assigned/reserved, is absent from the effective unassigned pool and normal CandidateLists, cannot be assigned to another pawn, and is not a `TrySteal` victim item. The future event/integration layer owns the concrete executing-reservation representation and decides when completion, interruption, cancellation or invalidation releases or removes that ownership. The caller/event-integration layer supplies a route-start context (`InitialPosition`, `StartTime`) that already reflects the point from which the first still-planned step will begin. Route Planner relies on these invariants and does not define how the execution layer maintains them.

A planned step stores route-local execution state and locally composable snapshot metrics:

PlannedStep  
{  
WorkItem  
PlannedWorkDuration // cached scheduled duration; partial-step execution budget  
PlannedReward // cached reward for this planned execution  
TravelFromPrevious // cached travel duration from route origin/previous step  
CompletesWorkItem // cached completion prediction for this snapshot  
MustRemainAssigned  
}

PlannedWorkDuration and CompletesWorkItem are cached execution estimates in the current route snapshot. For a full step, PlannedWorkDuration caches the estimated full work duration and CompletesWorkItem is true while that full step remains valid. For a partial step, PlannedWorkDuration is also the current execution budget and CompletesWorkItem is the current prediction for that budget. PlannedReward and TravelFromPrevious are cached snapshot metrics. Structural route mutations update affected local values and route aggregates; externally caused estimate changes are maintained event-driven by the integration layer.

MustRemainAssigned is a route-maintenance/optimization invariant supplied by higher-level planning. Within route-preserving operations the item may move atomically but may not be dropped. If the caller performs a full structural discard/rebuild, surviving protected work may be carried into RequiredJobs according to Work Planner semantics; Route Planner does not decide why the protection exists and does not infer MustRemainAssigned merely because an item appears in RequiredJobs. Any protection that survives such a rebuild is higher-level Work Planner bookkeeping on the rebuilt result.

For partial execution, the current PlannedWorkDuration is an absolute work-time budget, not a completion fraction. If an explicitly handled state change alters work speed/remaining work, event-driven maintenance may update PlannedWorkDuration, CompletesWorkItem and dependent reward/result-position metrics. Such refresh must not silently convert a previously valid full step into an unsupported or unplanned partial step; if the route constraints no longer support the step, the caller repairs or invalidates the route. Zero-work items are instantaneous and never partial. v1 planner/simulator assumes elementary WorkItems; complex interaction-cell selection, clustering/packages and moving-work decomposition remain deferred integration concerns.

Whenever a Route Planner operation changes a partial step's `PlannedWorkDuration`, that mutation atomically refreshes every route-snapshot value whose meaning depends on the duration. At minimum this includes the step's `PlannedReward`, `CompletesWorkItem` and resulting position through `EndPosition(worker, workDuration)`; any affected following `TravelFromPrevious`; route `EndPosition` when affected; WorkDuration, WalkingDuration, TotalDuration, TotalReward and Score/Q aggregates; and terminal-travel feasibility. Unaffected steps and unrelated local metrics are not defensively reevaluated. This contract applies both when `MaximizePartial` increases a budget and when compression reduces one.

# 3. Planner services, travel abstraction and caching boundary

Route search owns no RimWorld-specific pathing, reward model or Primary/Backup rules. A RoutePlanner instance receives long-lived services at construction:

RoutePlanner(  
ITravelProvider travelProvider,  
IRewardProvider rewardProvider,  
IAssignmentEligibilityProvider eligibilityProvider)

ITravelProvider  
{  
GetTravelTime(worker, from, to) // signed duration  
}  
  
return \>= 0 =\> travel duration  
return \< 0 =\> no path

Within one fixed topology/pathing snapshot, `GetTravelTime(worker, from, to)` returns the minimum traversable elapsed travel time between those endpoints. Therefore, whenever `A -> X` and `X -> B` are reachable in the same snapshot, their concatenation proves `A -> B` reachable and the directed shortest-time inequality holds:

```text
T(A, B) <= T(A, X) + T(X, B)
```

This minimum-travel contract is the canonical basis for v1 compression/steal reasoning that removing a stationary elementary step cannot by itself make the bridge between its retained neighbors slower than travelling through that removed step. In v1, `CompressRoute` and `TrySteal` require **removal-safe stationary WorkItems**: removing such a step does not itself provide movement, change topology/pathing, or otherwise alter the retained route's endpoint semantics beyond deleting that step. WorkItems whose execution provides movement across otherwise disconnected space, changes topology, or has other removal-sensitive movement semantics are not valid inputs to those two operations until those semantics are designed explicitly.

IRewardProvider supplies already-computed worker/work-item Reward and owns both full and partial execution values. Route search does not derive Reward from duration, UI Priority, skill/result formulas or CompletionTravelBonusCells.

IRewardProvider  
{  
GetReward(worker, IWorkItem item)  
GetPartialReward(worker, IWorkItem item, workDuration)  
}

Route Planner makes no assumptions about how previous partial progress changes later full/partial Reward. The current IWorkItem state plus IRewardProvider fully define that value.

IAssignmentEligibilityProvider  
{  
CanAssign(worker, route, IWorkItem newItem)  
CanUnassign(worker, route, IWorkItem item)  
CanSteal(receiverWorker, receiverRoute,  
victimWorker, victimRoute,  
IWorkItem stolenItem,  
IWorkItem receiverReplacedItem = null)  
}

The planning/context infrastructure versions the **complete colony planning state**, not merely the changes created by the current planning session. `ColonyStateContext` is the authoritative context representing the current real planning state of the colony: effective planned routes, assigned/reserved ownership, the unassigned work pool, `MustRemainAssigned` flags and other planning facts that have been materialized into real colony planning state. Assigned/reserved ownership includes both planned assignment represented by a PlannedStep and executing reservation represented outside PlannedRoute. Route Planner redistribution operates only on planned assigned work present in supplied routes; executing reservations are not route members and are never normal assignment/steal candidates. Ordinary planner operations may read this context but never mutate it.

A Work Planner session creates `RootSessionContext` as an unchanged child of `ColonyStateContext`. It initially has no session-local delta and therefore observes exactly the current real colony planning state. v1 assumes the session is atomic with respect to external mutation of ColonyStateContext. All planning alternatives live in `RootSessionContext` or descendants. Only Work Planner `FinalGlobalCommit` may atomically materialize the final persistent session result back into `ColonyStateContext`; Route Planner never performs that merge.

Every route participating in a planning operation is associated semantically with an opaque `ContextId`. The eligibility provider resolves that context and its ancestors to obtain the full effective route/assignment state, the context-versioned effective unassigned work pool and coordinated-planning obligations. A route visible in a child context is the complete effective route for that pawn in that alternative state, regardless of which earlier planning session originally created any individual step. No planning-session provenance or old/new-step distinction affects route semantics.

For every unresolved planning pawn the effective context also carries `HasPlannedPrimary` and `PrimaryCoverageRelaxed`:

PlanningPawnState  
{  
Pawn  
HasPlannedPrimary // cached: true iff this pawn's complete effective PlannedRoute in this context contains >= 1 Primary  
PrimaryCoverageRelaxed // context-local; exclude this unresolved pawn from coordinated matching for the current decision  
}  
  
PlanningContext  
{  
ContextId  
ParentContextId  
PlanningPawns // unresolved PlanningPawnState entries, when this context is part of a planning session  
RequiredPrimaryCoverage // fixed optimistic target in normal coordinated mode; refreshed after relaxed decision finalization  
AssignmentDelta // conceptual; concrete full-state/delta representation is deferred  
CandidateLists // immutable ListId -> work-item snapshot  
RequiredLists // immutable ListId -> required-item snapshot  
}

`HasPlannedPrimary` is a context-local cache of a property of the **whole effective route**, not a separate ownership/provenance mechanism. Any accepted mutation that can change whether the whole effective route contains Primary work — assignment, ordinary release, transfer, hard invalidation, compression/rebuild or another accepted route-membership change — must leave that cache consistent with the resulting effective route. The concrete invalidation/recomputation mechanism is deferred to the context-layer design.

Assigning an item consumes it from the effective unassigned pool; ordinary standalone unassignment returns it. Hard invalidation is different: an item that is destroyed, already completed elsewhere or otherwise no longer part of the planning universe is removed from assignment/planning state and is not returned to the pool. Child contexts inherit their parent's complete effective state and apply their own changes. Context lifetime is infrastructure-owned, so deferred search/steal candidates may safely retain the effective state they were built against.

Independently compared Route Planner answers are isolated in child operation contexts created from the relevant Work Planner context. Route Planner may create further descendants for internal search. Alternatives must not mutate sibling or parent state. Before a Route Planner operation returns, losing internal alternatives are discarded and the selected internal state is merged into the context supplied for that operation. CandidateLists/RequiredLists are immutable snapshots used to bound a particular planner call and are not authoritative ownership state; a changed candidate snapshot gets a new ListId.

Temporary candidate/branch route snapshots are an expected part of planning, but this document intentionally does not prescribe whether complete routes or accepted route changes are stored directly inside contexts, beside them, through immutable handles, wrappers, deltas or another context-layer representation. References to a route being “associated with” a context are semantic requirements, not a prescribed mutable `ContextId` API. Concrete route/context binding, full-state/delta storage, copy/merge mechanics, lifetime management and pooling remain deferred to the dedicated context-layer design.

CanAssign first enforces hard assignment eligibility, including that `newItem` is currently unassigned in the effective context, then applies the coordinated coverage rule selected by that context.

**Normal coordinated mode — no unresolved pawn is relaxed.** `RequiredPrimaryCoverage` is supplied by Work Planner as the fixed optimistic target for the current unresolved set; at coordinated-root initialization it is `coveredRoot + MaximumPrimaryMatching(uncoveredRoot, effectiveUnassignedPool)`. Let `covered` be the non-relaxed unresolved PlanningPawns whose prospective `HasPlannedPrimary` is true, `uncovered` the remaining non-relaxed unresolved pawns, and:

```text
requiredFromMatching = max(
    0,
    RequiredPrimaryCoverage - |covered|)
```

The prospective mutation is coverage-safe only when:

```text
MaximumPrimaryMatching(
    uncovered,
    effectiveUnassignedJobsAfterMutation)
>= requiredFromMatching
```

Multiple Primary items for one pawn still contribute only one covered slot. Already resolved pawns are absent from PlanningPawns.

**Relaxation mode — at least one unresolved pawn has `PrimaryCoverageRelaxed = true`.** Relaxed pawns are excluded from both covered and matching sets. The old fixed target is not used for intermediate eligibility. Instead the mutation must preserve the current maximum achievable coverage of the non-relaxed unresolved pawns:

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

This dynamic rule deliberately captures coverage opportunities created during the same decision, including a receiver replacement that releases a new Primary item into the pool. Any later CanAssign in the same effective context may not consume that opportunity if doing so would reduce the best coverage currently achievable for the remaining non-relaxed pawns.

**Non-coordinated fast path.** Singleton and ordinary local-repair contexts may still contain their one affected PlanningPawn but carry `RequiredPrimaryCoverage = 0` and do not enter relaxation mode. Their coordinated requirement is vacuously satisfied, so CanAssign may skip maximum matching entirely. This fast path must not be based only on `PlanningPawns.Count < 2`, because a coordinated context may have one unresolved pawn with a non-zero obligation.

A reverse Primary index remains an additional fast path only where the effective coverage mode proves that the prospective mutation cannot affect either relevant matching edges **or covered/uncovered membership**. It is not sufficient merely to prove that the item is not Primary for some other pawn: a mutation that changes the current pawn's `HasPlannedPrimary` can also change the coverage equation and therefore may still require matching reevaluation.

CanUnassign is used only for an ordinary release of a still-valid assigned item. The item returns to the effective unassigned pool. The prospective context state updates `HasPlannedPrimary` from the resulting **whole effective route**; removing one Primary leaves the flag true when another Primary remains anywhere in that route and clears it only when the last Primary is removed. Coverage is then evaluated under the same effective mode as CanAssign: fixed `RequiredPrimaryCoverage` in normal coordinated mode, or `coverageAfter >= coverageBefore` over non-relaxed unresolved pawns in relaxation mode. Local backing still applies independently, and if the removed item is Primary for this worker, generic v1 eligibility permits removal only when another Primary item for that worker remains in the route. Hard invalidation bypasses this ordinary-release contract because an invalid/nonexistent item must leave the planning universe rather than re-enter the pool, but the accepted invalidation still updates `HasPlannedPrimary` from the resulting effective route. A different case — an otherwise globally valid WorkItem whose assignment becomes invalid specifically for this pawn because of policy/capability/allowed-area/accessibility changes — is intentionally outside the v1 Route Planner mutation contract and belongs to the future event/integration layer; it should not be forced through either CanUnassign or hard work-item invalidation. The local-backing rule intentionally preserves the documented rare inherited-recovery repeated-repair limitation.

CanSteal validates an atomic transfer of already-assigned work. For pure insertion it evaluates stolenItem moving directly from victim to receiver. When receiverReplacedItem is supplied, that receiver PlannedStep must not have MustRemainAssigned; the prospective state is stolenItem: victim -> receiver and receiverReplacedItem: receiver -> unassigned pool. A protected receiver step is therefore simply not eligible for replacement during steal. The transferred stolen item never enters the unassigned pool, and receiver assignment may ignore existing ownership only when that ownership is exactly the declared victim. CanSteal applies the relevant hard ownership/capability rules and local Primary-backing rules to the final transfer state. The transfer itself does not repeat the coordinated matching check used when consuming an item from the unassigned pool: the stolen item was already absent from that pool, and local Primary-backing rules prevent an accepted transfer/replacement from removing the last Primary from an affected Primary-backed route. A released receiver item only returns work to the effective unassigned pool and therefore cannot by itself reduce currently achievable coordinated coverage. The accepted prospective transfer state updates `HasPlannedPrimary` for every affected unresolved pawn from its resulting complete effective route. Any later operation that consumes work from the pool passes the ordinary CanAssign coverage checks in that then-current effective context. If the victim PlannedStep containing stolenItem has MustRemainAssigned, the flag moves with that step and the transfer may not leave it unassigned. MustRemainAssigned permits an atomic move only when ordinary CanSteal eligibility, including local Primary backing, also permits that transfer; it is not a guarantee that every protected item is transferable in every route state.

The future event/integration layer guarantees that `ColonyStateContext` represents the current real colony planning state for every explicitly supported event/change. `PlannedRoute` contains only not-yet-started work; executing work is outside it. Work/Route Planner documents intentionally specify these state guarantees rather than the operations, ordering or storage mechanisms by which the integration layer maintains them. Route Planner does not receive raw game state and does not defensively synchronize it; it operates on the effective state of the supplied planning context plus operation-specific inputs derived from that state.

Long-lived caching belongs primarily inside ITravelProvider because path calculation is expected to dominate simple Reward/s arithmetic. Short-lived candidate memoization is allowed; persistent arbitrary route-combination caching is deferred.

# 4. Route score

For every worker/work-item evaluation, the planner obtains full or partial Reward from IRewardProvider. It does not calculate UI Priority, quality models, completion-travel bonus or Primary/Backup weighting itself.

ScoreDuration(route) = max(TotalDuration(route), OnePlannerTimeQuantum)\
Q(route) = TotalReward(route) / ScoreDuration(route)\
  
TotalDuration = WorkDuration + WalkingDuration  
WalkingDuration = sum of planned route travel legs  
WorkDuration = sum of PlannedWorkDuration  
  
Q(empty route) = 0  
TotalReward(empty route) = 0  
WorkDuration(empty route) = 0  
WalkingDuration(empty route) = 0  
TotalDuration(empty route) = 0

For an operation-local empty route snapshot, `EndPosition = InitialPosition` for that operation. This makes terminal-travel feasibility well-defined when removal/steal eliminates the last planned step. Empty route snapshots are valid transient inputs/results for operations that preserve or repair existing routes; persistence/Idle semantics belong to Work Planner.

`OnePlannerTimeQuantum` is the canonical smallest positive duration unit used by planner inputs (one simulation tick in the v1 simulator). Travel and work durations are non-negative integral multiples of that quantum after provider conversion, so the denominator substitution changes only an exactly zero-duration route. Consequently a zero-duration/zero-Reward route has `Q = 0`, a positive-Reward zero-duration route has the finite score `TotalReward / OnePlannerTimeQuantum`, and ordinary finite arithmetic — including anchor-regret subtraction — is well-defined. Multiple instantaneous items accumulate Reward normally rather than producing IEEE infinity or NaN.

GetWorkTime(worker) = 0 is valid. If a candidate mutation adds positive Reward with zero additional elapsed duration, it is a strict improvement under the finite formula above.

Walking lowers Q through route elapsed time. Generic route ordering is centralized here rather than restated by individual algorithms.

## 4.1 Canonical route comparison

`CompareRoutes(A, B)` is the generic Route Planner comparator for two alternative routes after any operation-specific higher-priority constraints have already been applied:

```text
1. higher Q wins
2. if Q is equal, lower WalkingDuration wins
3. otherwise the alternatives are Equivalent
```

`Equivalent` means Route Planner has no generic preference. A higher-level caller may apply a policy-specific tie-break; otherwise either alternative may be kept. Algorithm sections refer to `CompareRoutes` instead of restating this ordering.

`CompareRoutePairs(A, B)` is the corresponding comparator for an affected two-route configuration used by cross-route optimization:

```text
1. higher sum of route Q wins
2. if equal, lower combined WalkingDuration wins
3. otherwise the configurations are Equivalent
```

Reserved terminal travel from the route result position to HorizonEndPosition participates in horizon feasibility but intentionally does not contribute to TotalDuration, WalkingDuration or Q. It is a reachability/safety reserve and is not necessarily executed after this route; charging it into every short route would repeatedly penalize travel that may occur only once.

# 5. Absolute time, horizon and TimeShift

A construction/normalization/augmentation request supplies the absolute `StartTime` and operation-specific horizon constraints chosen by its caller, including `Horizon`/`BaseHorizon`, `HorizonEndPosition`, and where applicable `MaxTimeShift`. `CompressRoute` instead receives one already-derived target `Horizon` plus `HorizonEndPosition`: it does not interpret TimeShift policy. Route Planner combines only those supplied constraints with cached/local route durations and terminal travel to test feasibility. Persistent PlannedRoute itself does not store StartTime, EndTime, Horizon, TimeShift or MaxTimeShift.

The canonical absolute-time feasibility definition is:

```text
TerminalTravel(route, HorizonEndPosition) =
    GetTravelTime(worker, route.EndPosition, HorizonEndPosition)

RequiredElapsed(route, HorizonEndPosition) =
    route.TotalDuration + TerminalTravel(route, HorizonEndPosition)

FitsHorizon(route, StartTime, Horizon, HorizonEndPosition) iff
    TerminalTravel >= 0
    and StartTime + RequiredElapsed(route, HorizonEndPosition) <= Horizon
```

For an operation-local empty route snapshot, `route.EndPosition = InitialPosition`, so the same formula applies without a special case. `RequiredElapsed` is a duration; `StartTime` and `Horizon`/`BaseHorizon` are absolute times. All BuildRoute, MaximizePartial, ExpandRoute, CompressRoute, TrySteal and required-search feasibility checks use this definition even when pseudocode omits repeated request-context arguments.

`MaxTimeShift` is always **relative to the Horizon/BaseHorizon supplied to that specific Route Planner operation**. For a candidate whose protected completion may use shift, the canonical required-shift calculation is:

```text
requiredShift = max(
    0,
    StartTime + RequiredElapsed(route, HorizonEndPosition) - BaseHorizon)

allow only if requiredShift <= MaxTimeShift
```

Unreachable terminal travel is infeasible rather than an arbitrarily large shift. A positive `requiredShift` remains legal only when the operation-specific protected-completion rules below authorize it. Route Planner does not calculate the caller's colony-level horizon policy, does not reconstruct constraints for later calls, and does not maintain any cross-operation horizon state.

TimeShift is operation-local and is best understood as a **bounded completion allowance**, not as general optional-work budget. Positive shift may be used only when the additional time crosses a completion boundary that the operation explicitly protects. It is never used merely to increase incomplete partial progress, improve Q or make an ordinary optional job fit.

For BuildRoute, the protected completion boundary is its non-empty RequiredJobs set under the detailed Section 7 rules. Ordinary construction with an empty RequiredJobs set never uses construction TimeShift.

For `MaximizePartial`, the protected completion boundary is full completion of the one already-existing partial PlannedStep. The operation first maximizes that partial inside BaseHorizon. It may use the **minimum necessary** positive shift only when doing so fully completes that WorkItem inside `BaseHorizon + MaxTimeShift`. If full completion is still impossible, positive shift is not used merely for additional partial progress.

For ExpandRoute, the only protected completion boundary is its optional single RequiredJob/anchor. Ordinary optional augmentation never creates additional shift. ExpandRoute does not itself maximize an existing partial; Work Planner invokes `MaximizePartial` explicitly at orchestration points that require a normalized baseline.

A successful operation returns a route plus only the operation-specific result metadata needed by its caller. For operations whose decision can legitimately introduce TimeShift, that metadata may include the amount of shift actually required/used; the exact C# result shape is deferred. Names such as `RequiredShift` or `UsedTimeShift` in conceptual pseudocode are illustrative placeholders rather than a finalized shared result API. Route Planner does **not** return a new Horizon and does not turn one operation's time result into the constraints of a later public call. The caller supplies later-call constraints according to its own higher-level time policy; Route Planner does not carry horizon state forward.

`CompressRoute` is different: the caller has already reduced higher-level timing policy to one target `Horizon`. Compression either returns a route that fits that supplied limit or returns `CannotFitProtectedRoute`; it has no TimeShift input, output or policy semantics.

# 6. BuildRoute, MaximizePartial, ExpandRoute, CompressRoute and result model

The v1 public route API distinguishes from-scratch construction, explicit normalization of an already-existing partial, augmentation of an existing route, simple bounded compression and cross-route redistribution. In normal Work-Planner policy dispatch, `BuildRoute` is used when the pawn has no route and `ExpandRoute` augments an existing route. `MaximizePartial` is never an implicit phase of ExpandRoute; Work Planner calls it explicitly when orchestration requires partial normalization. Existing route snapshots are assumed current through their planning context; event/integration maintenance is responsible for real-state synchronization.

BuildRouteRequest  
{  
Worker  
Route // MUST be empty; carries ContextId  
CandidateListId // immutable optional-candidate snapshot in context infrastructure  
RequiredListId // immutable required-item snapshot in context infrastructure  
InitialPosition  
StartTime // absolute operation context  
BaseHorizon // unshifted base absolute horizon\
HorizonEndPosition  
MaxTimeShift  
}

MaximizePartialRequest  
{  
Worker  
ExistingRoute // non-empty effective route; at most one partial step; normalization baseline, not a repair/compression input  
InitialPosition  
StartTime  
BaseHorizon  
HorizonEndPosition  
MaxTimeShift  
}

ExpandRouteRequest  
{  
Worker  
ExistingRoute // non-empty effective baseline in normal Work-Planner use; existing steps preserved  
CandidateListId // immutable unassigned optional-candidate snapshot resolved through route.ContextId  
RequiredListId // empty or exactly one required item/anchor  
InitialPosition  
StartTime  
BaseHorizon // caller-supplied horizon constraint for this augmentation call  
HorizonEndPosition  
MaxTimeShift // relative to BaseHorizon; normally 0 for ordinary augmentation, may be positive for a single required anchor  
}

CompressRouteRequest  
{  
Worker  
ExistingRoute // current snapshot; carries ContextId  
InitialPosition  
StartTime  
Horizon // caller-derived target limit that the resulting route must fit  
HorizonEndPosition  
}

PlannedRoute  
{  
Worker  
ContextId // opaque planning-context reference  
Steps[]  
EndPosition // cached resulting position for this snapshot  
TotalReward  
WorkDuration  
WalkingDuration  
TotalDuration  
Score // Q(route)  
}

`BuildRoute` has an explicit `Success` / `Failure` result contract and never returns `Success(empty)`.

- With `RequiredJobs == {}`, `Success` means at least one ordinary candidate WorkItem was scheduled as a full step. If no candidate can be scheduled, BuildRoute returns `Failure`.
- With `RequiredJobs != {}`, `Success` means at least one RequiredJob is represented in the route, either fully or by a valid positive-duration partial under Section 7. If no RequiredJob can be represented, BuildRoute returns `Failure`.
- On `Failure`, losing/speculative internal branches are discarded and the caller-supplied operation context is unchanged.

A successful BuildRoute result carries the route plus operation-local result metadata. For non-empty RequiredJobs this includes `RequiredSetSatisfied`, `CompletedRequiredJobs`, `IncompleteRequiredJobs` and, when useful to the caller, the shift actually required by the selected required configuration. Route Planner does not return a new Horizon.

BuildRoute requires Route.Steps to be empty. It resolves RequiredListId and CandidateListId through Route.ContextId. With an empty required snapshot it follows ordinary zero-shift seed/expansion rules and creates only full steps. With required items it uses the exhaustive Section 7/8 search. Before BuildRoute returns Success it destroys losing internal branches and merges the winning assignment state into the supplied operation context; before Failure it destroys all internal branches and leaves that context unchanged.

`MaximizePartial` preserves route membership and relative order. If no partial exists it returns Success with the unchanged route. If one partial exists, it applies the Section 12.1 normalization rule and returns the normalized route; operation metadata may report any positive shift actually required to complete the partial. It never decreases the existing partial `PlannedWorkDuration`; shrinking a partial budget belongs to `CompressRoute`, not `MaximizePartial`. It never assigns, unassigns or reorders a WorkItem, so no CandidateListId/RequiredListId is involved.

`ExpandRoute` augments an existing normalized route. `RequiredListId` must resolve to `{}` or exactly one item. With an empty required list, inability to improve the route is **not Failure**: returning the unchanged route is a normal Success. With one required item, Failure means only that this required augmentation could not be represented under the ExpandRoute contract; the pre-existing baseline route remains valid and the failed operation context is left unchanged.

A successful ExpandRoute result carries the resulting route and, when a RequiredJob was supplied, whether that required item was fully satisfied or represented by the allowed partial fallback; result metadata may also report any shift actually required by that protected required stage. ExpandRoute never extends an already-existing partial; its caller is responsible for invoking MaximizePartial when required by policy. The global route invariant remains at most one partial PlannedStep.

CompressRoute receives no candidate list and never adds work. It may reduce an existing partial budget and/or release unprotected existing steps until the route fits the supplied target `Horizon`. On success it returns only the compressed route state; compression has no shift-result metadata. The successful route may be empty if compression legally released the final unprotected step and the resulting empty route can still satisfy terminal reachability. On `CannotFitProtectedRoute` it leaves the caller-visible parent state unchanged under the documented child-operation transaction rule. `CannotFitProtectedRoute` is an umbrella compression-failure result: failure may arise from MustRemainAssigned protection, assignment eligibility/local Primary backing or the absence of a legal positive-gain compression step.

The BuildRoute prohibition on `Success(empty)` is operation-specific. Empty PlannedRoute snapshots may still exist transiently in compression or after a steal removes the final victim step. Work Planner assigns the higher-level orchestration meaning.

# 7. RequiredJobs and partial-execution semantics

RequiredJobs refers to the work items resolved from RequiredListId for the current Route.ContextId. These are construction constraints distinct from PlannedStep.MustRemainAssigned. Route Planner imposes no algorithmic size limit on RequiredJobs; v1 enumerates the complete policy-valid ordered-subset/permutation search space. If gameplay later produces sets large enough to make this too expensive, the search strategy will be revisited based on profiling.

For non-empty RequiredJobs, BuildRoute first tries to complete the whole set. If complete construction is possible, choose minimum necessary local requiredShift, then use `CompareRoutes`. If the complete set is impossible, use the exact incomplete-required fallback below rather than switching to a TotalReward objective. A successful result must represent at least one RequiredJob; an empty required configuration is search scaffolding only and can never be returned as `Success`.

## 7.1 Exact complete-RequiredJobs mode

Search every policy-valid permutation of the complete RequiredJobs set. Reject a mutation when CanAssign fails, GetWorkTime(worker) \< 0, or required travel is unreachable. A complete configuration may use shift only as needed to finish RequiredJobs and still reach HorizonEndPosition, never to create a partial tail. Exact-search pruning may use non-negative shortest-path/travel assumptions and the elementary IWorkItem movement contract.

Among routes containing every RequiredJob fully completed, choose minimum requiredShift; among equal-shift alternatives use `CompareRoutes`. If all required work fits the base Horizon, requiredShift is zero. This selected required configuration is the required baseline used by BuildRoute's optional-expansion rule in Section 7.3.

## 7.2 Incomplete RequiredJobs fallback

k0 = maximum fully completed RequiredJobs that still reach HorizonEndPosition inside base Horizon  
kMax = maximum fully completed RequiredJobs that still reach HorizonEndPosition inside BaseHorizon + MaxTimeShift

If kMax \> k0, shift may be used because it preserves more RequiredJobs. Choose a kMax route by minimum necessary requiredShift including terminal travel, then use `CompareRoutes`. The shifted work sequence ends immediately after a fully completed work item; shifted time may not be spent travelling to or partially executing another required item except for terminal travel. Because kMax \> k0, this branch necessarily represents at least one fully completed RequiredJob and therefore produces a successful required baseline.

If kMax == k0, requiredShift = 0. Positive shift is not allowed merely to improve Q, TotalReward, walking or partial progress. Among zero-shift routes completing k0 RequiredJobs, first identify all configurations with that maximum full-completion count.

When k0 \> 0, every such full configuration is a valid successful required baseline. Each is also evaluated with each remaining RequiredJob that has SupportsPartialExecution = true as a single terminal partial alternative. For each such alternative PlannedWorkDuration is deliberately fixed to the maximum feasible positive work duration that still leaves enough time for terminal travel to HorizonEndPosition inside the base Horizon. v1 does not search shorter partial durations for a better Q; this is a maximum-progress policy. The unmodified full-only configuration remains an alternative, so a partial tail is kept only when it wins `CompareRoutes`.

When k0 == 0, the empty full configuration is not a successful result. Enumerate every RequiredJob with SupportsPartialExecution = true as a single terminal-partial alternative using the same maximum-feasible positive-duration rule. If at least one valid positive-duration partial exists, choose the best partial alternative using `CompareRoutes` and return it as a successful incomplete required baseline. If no valid positive-duration partial exists, BuildRoute returns `Failure`.

Any successful incomplete baseline returns `RequiredSetSatisfied = false`, `CompletedRequiredJobs` for the fully completed required items, and `IncompleteRequiredJobs` for the remaining required items. Newly planned partial execution never consumes MaxTimeShift.

## 7.3 Optional expansion after a required baseline

`RequiredSetSatisfied` does not decide whether optional work may be added. After **any** successful required baseline — complete or incomplete, zero-shift or positive-shift — BuildRoute may run ordinary full-work optional augmentation through the private/internal optional-augmentation helper defined in Section 11. This is an internal phase of the same `BuildRoute` call, **not** a nested public `ExpandRoute` invocation.

Within one BuildRoute call, the required stage may establish a positive `requiredShift`. The subsequent internal optional-augmentation phase may then use the call-local limit `BaseHorizon + requiredShift`, with no authority to increase that limit further. Optional work may use capacity already available inside that call-local limit, including zero-duration or geometry-neutral improvements, but it may not introduce additional shift merely because RequiredJobs justified the required-stage extension.

This is the shift invariant inside Route Planner: positive shift may be introduced only by an explicitly protected completion boundary. How a later, separate public Route Planner call chooses its own Horizon and MaxTimeShift is entirely the caller's responsibility.

Review guardrail: an original RequiredJob omitted or left incomplete by the exact required stage cannot later become a newly completed ordinary optional insertion within that same `BuildRoute` call. If such a full insertion were feasible under the same call-local limit, the exact RequiredJobs search would have admitted a configuration with the corresponding greater full-required completion count. This is therefore not a reason to add RequiredJob-provenance filtering to the ordinary optional-candidate helper.

## 7.4 Partial execution contract

BuildRoute-created partial execution remains a terminal fallback only for an incomplete RequiredJob whose WorkItem has `SupportsPartialExecution = true`. Here “terminal” means terminal within the RequiredJobs baseline search: subsequent ordinary optional augmentation may insert full steps before or after that partial, provided the final route still satisfies the global at-most-one-partial invariant. Ordinary BuildRoute optional work is never partial. For each eligible partial RequiredJob, BuildRoute evaluates exactly one zero-shift partial duration: the maximum feasible positive execution duration inside the base Horizon after reserving terminal travel.

`MaximizePartial` owns later progression/completion of an already-existing partial PlannedStep. ExpandRoute never changes the budget of that existing partial. Its only partial-related responsibility is the single-RequiredJob fallback: when the normalized baseline has no partial, the required anchor may become the route's one partial according to Section 12.

A route must contain at most one partial PlannedStep after every operation. Therefore ExpandRoute returns Failure rather than creating a required partial when the supplied baseline already contains an incomplete partial.

A planned partial step uses its current cached PlannedWorkDuration as an absolute execution budget. IRewardProvider.GetPartialReward(worker, item, workDuration) and WorkItem.EndPosition(worker, workDuration) interpret that duration according to the concrete family. Event-driven maintenance may update the cached budget/completion prediction when explicitly handled state changes occur; execution uses the current maintained snapshot.

Caller contract: a WorkItem may expose SupportsPartialExecution = true only when the integration layer can persist unfinished work and provide correct partial reward/progress/result-position semantics. After a planned partial budget executes, the integration layer commits durable progress and returns any unfinished remainder to the ordinary unassigned pool; MustRemainAssigned on the executed step does not create a persistent completion obligation.

# 8. Exact search for RequiredJobs

For RequiredJobs, v1 exhaustively enumerates all policy-valid ordered subsets/permutations. For zero-shift configurations at the selected maximum full-completion count it also enumerates supported remaining RequiredJobs as single terminal-partial alternatives, each using that item's one maximum-feasible positive duration. The same search determines complete-set feasibility, maximum full-required completion at base/shifted limits and the best zero-shift partial fallback. This exact primitive does not make arbitrary optional-job construction globally exact. Each ordered-subset/permutation branch whose assignments differ is represented by an internal descendant context under the BuildRoute operation context; losing branches are released and the winning branch is merged back into the operation context only when BuildRoute ultimately succeeds.

**v1 eligibility scope guardrail.** The current multi-`RequiredJobs` exact-search guarantee is used by Work Planner for non-coordinated inherited-recovery construction, where coordinated Primary coverage is disabled (`RequiredPrimaryCoverage = 0`). The search currently applies ordinary assignment eligibility while growing each ordered branch. Do not reinterpret this as a guarantee of exact final-state search under arbitrary order-sensitive coordinated-coverage eligibility. If multi-required construction is later introduced into a coordinated context with a non-zero coverage obligation, the design must explicitly choose whether required-configuration eligibility is sequential-prefix-based or evaluated atomically against the prospective final required configuration before claiming exhaustive coordinated semantics.

```text
BuildRequiredRouteExact(requiredJobs):
    baseFullCandidates = SearchAllOrderedRequiredSubsets(
        requiredJobs, BaseHorizon, HorizonEndPosition)
    maxFullCandidates = SearchAllOrderedRequiredSubsets(
        requiredJobs, BaseHorizon + MaxTimeShift, HorizonEndPosition)

    if maxFullCandidates contains a route completing all RequiredJobs:
        best = all-complete candidate with:
            minimum requiredShift from base Horizon
            then CompareRoutes
        return RequiredBaseline(
            Route = best,
            RequiredShift = requiredShift(best),
            RequiredSetSatisfied = true,
            CompletedRequiredJobs = all RequiredJobs,
            IncompleteRequiredJobs = {})

    k0   = maximum fully completed RequiredJobs in baseFullCandidates
    kMax = maximum fully completed RequiredJobs in maxFullCandidates

    if kMax > k0:
        best = candidate completing kMax RequiredJobs with:
            minimum requiredShift including terminal travel
            then CompareRoutes
        return RequiredBaseline(
            Route = best,
            RequiredShift = requiredShift(best),
            RequiredSetSatisfied = false,
            CompletedRequiredJobs = fully completed required items in best,
            IncompleteRequiredJobs = all remaining required items)

    // Here kMax == k0 and positive shift is not used.
    zeroShiftFull = every baseFullCandidate completing k0 RequiredJobs

    if k0 > 0:
        alternatives = zeroShiftFull
        for each route in zeroShiftFull:
            for each incomplete RequiredJob R with R.SupportsPartialExecution:
                if a positive-duration terminal partial R fits inside base Horizon:
                    add route + maximum-feasible terminal partial R to alternatives
        best = candidate from alternatives by CompareRoutes
        return RequiredBaseline(
            Route = best,
            RequiredShift = 0,
            RequiredSetSatisfied = false,
            CompletedRequiredJobs = fully completed required items in best,
            IncompleteRequiredJobs = all remaining required items)

    // k0 == 0: an empty full configuration cannot be returned as Success.
    partialAlternatives = empty
    for each RequiredJob R with R.SupportsPartialExecution:
        if a positive-duration terminal partial R fits inside base Horizon:
            add maximum-feasible terminal partial R to partialAlternatives

    if partialAlternatives is empty:
        return Failure

    best = partial alternative by CompareRoutes
    return RequiredBaseline(
        Route = best,
        RequiredShift = 0,
        RequiredSetSatisfied = false,
        CompletedRequiredJobs = {},
        IncompleteRequiredJobs = all RequiredJobs)
```

BuildRoute then applies its common successful-required-result rule:

```text
required = BuildRequiredRouteExact(requiredJobs)

if required is Failure:
    return Failure

route = required.Route
optionalAugmentationHorizon = BaseHorizon + required.RequiredShift

if CandidateListId is non-empty:
    route = AugmentRouteWithOptionalCandidates(
        ExistingRoute = route,
        CandidateListId = CandidateListId,
        HorizonLimit = optionalAugmentationHorizon,
        HorizonEndPosition = HorizonEndPosition)

return Success(
    Route = route,
    UsedTimeShift = required.RequiredShift,
    RequiredSetSatisfied = required.RequiredSetSatisfied,
    CompletedRequiredJobs = required.CompletedRequiredJobs,
    IncompleteRequiredJobs = required.IncompleteRequiredJobs)
```

# 9. Anchor-targeted use

Anchor-targeted is not a separate global solver. The higher-level caller has already selected a policy-valid pawn/anchor pair.

- If the pawn has **no existing route**, Work Planner calls BuildRoute with `RequiredJobs = {anchor}` plus the selected optional candidate pool.
- If the pawn already has an effective route, Work Planner calls ExpandRoute with `RequiredJobs = {anchor}` plus the selected optional candidate pool.

For BuildRoute, singleton required-set semantics remain those of Sections 7–8: every Success represents the anchor fully or through the permitted positive-duration partial fallback; unrelated work never substitutes for the missing anchor.

For ExpandRoute, Section 12 defines the single-required-item augmentation contract. Failure means this anchor cannot be represented in the existing route under that contract; it does not invalidate the baseline route.

In current Work Planner orchestration the anchor is Primary for the selected pawn. If an anchor-targeted candidate ultimately becomes the selected pawn decision, Work Planner marks the anchor PlannedStep `MustRemainAssigned = true`; generic Route Planner does not infer that flag merely from RequiredJobs.

# 10. Best-local use

Best-local supplies no RequiredJobs. When the pawn has no route, Work Planner invokes BuildRoute and the seed-selection algorithm below applies. When the pawn already has a normalized effective route, best-local is ordinary ExpandRoute with an empty RequiredListId; no BuildRoute seed selection is involved. Work Planner enforces Primary-first policy by selecting the candidate pool supplied to the appropriate primitive.

## 10.1 Seed 1 - best intrinsic work rate

F0 = { j \| a single-job route for j passes CanAssign, GetWorkTime(worker) \>= 0, is reachable, fully completes, and can still reach HorizonEndPosition inside the base horizon }  
Fzero = { j in F0 \| GetWorkTime(worker) == 0 }  
Fpositive = { j in F0 \| GetWorkTime(worker) \> 0 }  
  
Every item in Fzero becomes an additional zero-work seed. These seeds are added independently of the two ordinary heuristics below so instantaneous work is not hidden by duration-based ranking.  
  
If Fpositive is non-empty:  
j1 = argmax\_{j in Fpositive} R_j / WorkTime_j  
else:  
j1 = none  
  
If F0 is empty, BuildRoute returns `Failure`; there is no construction TimeShift or partial fallback.

Seed 1 ignores initial walking in its primary score. If primary values tie, prefer less initial walking. Zero-work items never enter this division; they are already represented by their dedicated additional seeds.

## 10.2 Seed 2 - best initial route rate

If Fpositive is non-empty:  
j2 = argmax\_{j in Fpositive} R_j / (TravelTime(InitialPosition, Start_j) + WorkTime_j)  
else:  
j2 = none

Seed 2 accounts for current position. If primary values tie, prefer less walking. If j1 and j2 are the same item, create only one ordinary seed. Zero-work items are still added separately even when one would also be attractive by initial route rate; this avoids both divide-by-zero and heuristic starvation of instantaneous work.

## 10.3 Seed expansion and selection

Expand every seed independently through the Section 11 private optional-augmentation helper under the fixed zero-shift base Horizon: up to two ordinary positive-work heuristic seeds plus every schedulable zero-work seed. Independently expanded seeds represent alternative assignment states and therefore use sibling internal child contexts under the BuildRoute operation context. Because seeds are created only from schedulable single-job routes, an unschedulable high-scoring job cannot hide a lower-scoring schedulable candidate. After comparison, losing seed contexts are released and the selected seed branch is merged back into the BuildRoute operation context. Return `Success` with the best expanded seed according to `CompareRoutes`; if the alternatives are Equivalent, either may be kept.

## 10.4 No construction TimeShift or optional partial fallback

For the no-route BuildRoute path, best-local has RequiredJobs = {} and therefore never uses construction TimeShift and never creates partial work. A candidate that cannot fully complete and still reach HorizonEndPosition inside the base Horizon is not a seed, even if it could fit within MaxTimeShift. If no seed exists, BuildRoute returns `Failure` to its caller.

For the existing-route ExpandRoute path, RequiredListId is empty. ExpandRoute uses the caller-supplied horizon constraints with no positive shift authority for ordinary optional work and returns the unchanged route successfully when no candidate improves it.

# 11. Private ordinary optional-augmentation helper

`AugmentRouteWithOptionalCandidates` is a private/internal Route Planner helper used by both `BuildRoute` and the public `ExpandRoute`. It performs only ordinary full-work greedy augmentation inside a horizon limit already established by its caller. It has no RequiredJob stage and no authority to introduce additional TimeShift. Calling this helper is not a separate public Route Planner operation.

Given a current policy-valid route and one candidate X resolved from the operation CandidateListId, first require Eligibility.CanAssign(worker, route, X). Then obtain `workTime = X.GetWorkTime(worker)`; if `workTime < 0`, reject X. Test X at every insertion position, including before the first and after the last existing step. Any layout whose newly required travel leg returns a negative duration is unreachable and is rejected. `CanAssign` does not replace execution-time or path-feasibility validation.

route: A -\> B -\> C  
  
X -\> A -\> B -\> C  
A -\> X -\> B -\> C  
A -\> B -\> X -\> C  
A -\> B -\> C -\> X

For insertion between A and B, the local time delta is conceptually:

DeltaT = Travel(A.end, X.start)  
+ WorkTime(X)  
+ Travel(X.end, B.start)  
- Travel(A.end, B.start)

The implementation creates a new PlannedStep snapshot for X, replaces only the affected adjacent travel legs, updates EndPosition when necessary and updates aggregate TotalReward/WorkDuration/WalkingDuration/TotalDuration/Score from those local deltas. It then verifies full-route feasibility including reserved terminal travel to HorizonEndPosition. Unaffected step reward/work/travel snapshots are not defensively recomputed.

For one helper iteration, evaluate every currently available candidate not already present in the route and keep its best feasible insertion layout. Among all such candidate/layout results, accept only the best route that **strictly beats** the current route according to `CompareRoutes`; positive Reward at zero marginal duration remains a strict improvement. Apply that assignment mutation inside the helper's operation context and repeat from the updated route. Stop when no ordinary candidate produces a strict improvement.

Conceptually:

```text
AugmentRouteWithOptionalCandidates(route, candidates, horizonLimit):
    currentRoute = route

    loop:
        best = currentRoute

        for each available candidate X not already in currentRoute:
            evaluate every insertion position using the rules above
            keep X's best feasible layout
            if that layout strictly beats best by CompareRoutes:
                best = layout

        if best == currentRoute:
            return currentRoute

        apply best assignment mutation to the helper operation context
        currentRoute = best
```

The concrete helper may reuse internal child contexts for competing assignment states exactly as other Route Planner search does; only its semantic greedy behavior is fixed here.

# 12. MaximizePartial and ExpandRoute v1

`MaximizePartial` and `ExpandRoute` deliberately have separate responsibilities. MaximizePartial normalizes the one already-existing partial step without adding or removing work. ExpandRoute assumes its existing route baseline has already been normalized whenever Work Planner policy requires that property, and deals only with required/optional augmentation.

## 12.1 MaximizePartial

`MaximizePartial` is a normalization primitive, not a repair or compression primitive. It only attempts to increase the existing partial budget; it never shrinks a `PlannedWorkDuration`, removes work or otherwise repairs an overlong route. If the supplied route already exceeds one or both supplied horizon bounds, that does not make the call invalid by itself: any attempted increase still has to satisfy the rules below, and if no legal increase exists the route is returned unchanged. `Success` therefore means that the normalization primitive completed normally, **not** that the entire input route has been proven horizon-valid; admission/maintenance/repair of the baseline remains caller-owned.

If the route has no partial PlannedStep, return the unchanged route with zero shift used by this operation.

If the route contains the one allowed partial PlannedStep P, its current `PlannedWorkDuration` is a lower bound for this operation: `MaximizePartial` never shrinks it.

1. starting from P's current budget, determine whether a larger `PlannedWorkDuration` can still keep the route terminally reachable inside `BaseHorizon`; if so, extend P to the maximum such BaseHorizon duration; if the current route already requires time beyond BaseHorizon, leave the current budget unchanged at this step rather than shrinking it;
2. if P still remains incomplete, test whether the **full remaining execution** can fit inside `BaseHorizon + MaxTimeShift`;
3. when full completion is possible, extend P exactly to completion and use only the minimum necessary positive shift;
4. when full completion is impossible even with MaxTimeShift, keep the larger of the original budget and any BaseHorizon extension found above, and use no positive shift merely for extra incomplete progress.

This is a normalization policy, not a Q comparison against another WorkItem. The operation never changes route membership or ordering. A local/pre-decision caller may supply positive `MaxTimeShift`; the same completion-boundary rule applies as anywhere else: positive shift is legal only when the minimum necessary extension fully completes P. If P completes, it becomes a normal full/completing PlannedStep and the route has no partial. Otherwise it remains the route's one partial.

Every accepted budget change applies the partial-budget metric-refresh contract from Section 2 before feasibility and result metadata are finalized. In particular, duration-dependent result position and any affected downstream/terminal travel are updated together with reward and route aggregates.

## 12.2 ExpandRoute single required item stage

If `RequiredListId == {}`, skip this stage.

Otherwise it must contain exactly one item R. ExpandRoute tests every legal insertion position for **full R** against the supplied baseline. Every tested assignment passes CanAssign and local metric/horizon validation.

If full R fits inside the currently authorized `BaseHorizon`, choose the best full insertion using `CompareRoutes`.

If full R does not fit there, it may use positive shift only when the minimum required extension fully completes R and the final route remains within `BaseHorizon + MaxTimeShift`. Among full R variants choose minimum necessary total shift, then use `CompareRoutes`.

If no full R variant is possible:

- when the baseline already contains a partial, return `Failure`; v1 never creates a second partial and ExpandRoute never shrinks or rebalances the existing partial to make room for R;
- when the baseline has no partial and `R.SupportsPartialExecution == true`, evaluate exactly one **zero-additional-shift** partial R variant at each insertion position: the maximum feasible positive duration inside the currently authorized BaseHorizon after reserving terminal travel. Choose the best partial-R route using `CompareRoutes`. If no positive partial R fits, return `Failure`;
- otherwise return `Failure`.

A successful required stage reports whether R is fully completed or remains the route's one partial required step.

## 12.3 Ordinary optional augmentation

After a successful required stage inside the same public `ExpandRoute` call, `ExpandRoute` delegates ordinary candidate growth to the Section 11 private `AugmentRouteWithOptionalCandidates` helper. The helper receives that call's resulting internal limit (`BaseHorizon + requiredShift`) and has no authority to introduce any additional shift.

The helper may add only **full** candidate WorkItems. It never extends an existing partial and never creates a new partial. All insertion feasibility, local metric-update and `CompareRoutes` rules are authoritative in Section 11 and are not duplicated here.

## 12.4 Result and transaction semantics

With `RequiredJobs == {}`, ExpandRoute always returns Success for a valid existing baseline route. The result may be structurally unchanged if no optional candidate improves the route.

With `RequiredJobs == {R}`, Success means R is represented according to Section 12.2; Failure means the required augmentation cannot be represented. On required Failure, all speculative changes inside that operation are discarded and the caller-supplied effective baseline remains unchanged.

ExpandRoute uses internal descendant contexts when tested assignment states differ and merges only its selected internal result into the supplied operation context. It never mutates ColonyStateContext directly.

The global route invariant after Success is: **at most one partial PlannedStep**.

# 13. CompressRoute

## 13.1 Compression goal and transaction boundary

`CompressRoute` is normally used when a current planned future route does not fit the target horizon supplied by its caller, or when some other event supplies a tighter target. It is deliberately a simple recovery fallback rather than an exhaustive local optimizer. The API is nevertheless total for an already-fitting input: it immediately returns that unchanged route before attempting any partial shrink or removal. It does not add work or drop `MustRemainAssigned` work. The request supplies one target `Horizon` and `HorizonEndPosition`. `ExistingRoute` is assumed current for its supplied `InitialPosition`/`StartTime` context.

The meaning of that target Horizon is entirely caller-owned. Work Planner may have derived it from sleep policy, maintenance constraints, allowed completion slack or other colony state, but CompressRoute does not know or interpret those reasons. Its only timing contract is that a successful returned route must fit the supplied Horizon including reserved terminal travel.

Before destructive removal, the operation first tests whether the unchanged route can fit by reducing an existing partial `PlannedWorkDuration` while keeping it positive. If so, it retains the maximum feasible partial budget and returns that compressed route.

Compression is executed inside a caller-created child operation context. Every accepted tentative removal and every resulting assignment-state change is confined to that operation context and its descendants; the parent context is never mutated directly. If compression returns `CannotFitProtectedRoute`, the operation context and its whole subtree are discarded, so the caller/parent planning state is exactly unchanged. On success, only the successful operation state may be accepted/merged according to the normal planning-context rules. This is a semantic transaction/isolation requirement; the concrete context API remains deferred.

If compression has removed every planned step, the fitting route is the valid transient empty route defined in Section 4: `TotalDuration = 0`, `WalkingDuration = 0`, and `EndPosition = InitialPosition`, so only terminal travel from that operation origin remains in the feasibility calculation. `Success(empty)` is therefore legal for CompressRoute; Work Planner decides that this ends route-preserving repair and whether to start from-scratch planning.

CompressRoute performs no post-compression expansion and returns no TimeShift/horizon metadata. Removed still-valid work is released to the effective unassigned pool inside the successful operation context; higher-level Work Planner may later expose those items again through policy-correct candidate snapshots passed to separate ExpandRoute calls.

## 13.2 Greedy removal compression

```text
CompressRoute(route, horizon, horizonEndPosition):
    currentRoute = route

    if FitsHorizon(currentRoute, horizon, horizonEndPosition):
        return Success(Route = currentRoute)

    loop:
        fitVariant = TryFitByReducingExistingPartial(
            currentRoute, horizon, horizonEndPosition)

        if fitVariant exists:
            return Success(Route = fitVariant)

        removalOptions = empty

        for each step S in currentRoute where S.MustRemainAssigned == false:
            if !Eligibility.CanUnassign(worker, currentRoute, S.WorkItem):
                continue

            candidateBase = RemoveAndUpdateLocalMetrics(currentRoute, S)

            candidateFit = TryFitByReducingExistingPartial(
                candidateBase, horizon, horizonEndPosition)

            timeGain = RequiredElapsed(currentRoute, horizonEndPosition)
                     - RequiredElapsed(candidateBase, horizonEndPosition)

            removalOptions += {
                Step = S,
                BaseRoute = candidateBase,
                FittingVariant = candidateFit,
                TimeGain = timeGain
            }

        if removalOptions is empty:
            return CannotFitProtectedRoute

        fittingOptions = all options where FittingVariant exists

        if fittingOptions is not empty:
            chosen = best fitting option by CompareRoutes(FittingVariant)

            ApplyChosenRemovalToEffectiveContext(chosen.Step)
            currentRoute = chosen.FittingVariant

            return Success(Route = currentRoute)

        improvingOptions = all options where TimeGain > 0

        if improvingOptions is empty:
            return CannotFitProtectedRoute

        chosen = improving option with:
            maximum TimeGain
            then CompareRoutes(BaseRoute)

        ApplyChosenRemovalToEffectiveContext(chosen.Step)
        currentRoute = chosen.BaseRoute
        // Keep the current structural route's partial budget unshrunk here.
        // The next iteration again tries the minimum shrink necessary to fit.
```

Each loop iteration evaluates all currently removable unprotected steps against the same current structural route without committing those hypothetical alternatives. For every single-removal candidate it also asks whether that structural removal **plus the minimum necessary shrink of the existing partial** would make the route fit. `TryFitByReducingExistingPartial` always retains the maximum positive partial budget that satisfies the supplied target Horizon.

If one or more single removals can make the route fit when combined with that minimal partial shrink, choose among those fitting variants by `CompareRoutes`. This prevents the fallback from unnecessarily deleting a second full job when one removal plus a smaller partial budget is already sufficient.

If no single removal can yet make the route fit even after allowed partial shrink, consider only removals with strictly positive `TimeGain`. `RequiredElapsed` means route `TotalDuration` plus reserved terminal travel to `HorizonEndPosition`, so the greedy choice reflects the real horizon effect rather than merely the removed WorkItem's own work duration. If no legal removal has `TimeGain > 0`, return `CannotFitProtectedRoute`: v1 does not accept a removal that leaves the fit unchanged or makes it worse in the hope that a later removal will compensate. The result name is an umbrella compression-failure outcome: failure may arise from `MustRemainAssigned`, eligibility/local-Primary-backing restrictions, or the absence of any legal positive-gain removal; it does not imply that a literal protected flag was necessarily the only blocker. Otherwise choose the positive-gain removal with the greatest `TimeGain`. The chosen structural baseline keeps its current partial budget unshrunk; the next iteration re-runs `TryFitByReducingExistingPartial`, preserving as much partial work as the new structure permits.

After a removal is selected, that ordinary release is applied only to the compression operation context, affected local route metrics are updated, and the next loop iteration starts from the selected route. `MustRemainAssigned` steps are never removal candidates, and `CanUnassign` may further exclude an unprotected step because of policy/local Primary-backing rules. If no legal positive-gain removal remains before the route fits, return `CannotFitProtectedRoute`; the whole failed operation context is discarded, leaving its parent unchanged. Higher-level Work Planner may then perform its documented full discard/rebuild from that unchanged parent state, with surviving protected work carried as inherited `RequiredItems`.

Under the Section 3 minimum-travel contract and its v1 removal-safe stationary-WorkItem precondition, removing an intermediate retained step does not by itself create an unreachable bridge or increase the minimum travel between its retained neighbors: the old two-leg path through the removed step remains a constructive reachable path in the same topology snapshot, while the direct provider result is no slower than that path. Likewise, removing a non-terminal step does not change the final retained step's result position. A WorkItem that violates the removal-safe stationary precondition is outside v1 `CompressRoute`, rather than a reason to add another reachability search layer here.

CompressRoute does not search replacements, combinations, swaps, DFS branches or post-compression insertions in v1. Its responsibility ends when it either returns a route that fits the supplied target Horizon or returns `CannotFitProtectedRoute`.

# 14. Existing order and reordering

v1 does not run a separate swap, relocate or 2-opt cleanup pass during ordinary construction/expansion. Each newly added item is tested at every insertion position, so a new item may appear before, after or between existing items. The relative order of already present items is otherwise preserved.

# 15. Cross-route steal / replacement optimizer

| Assignment invariant — MustRemainAssigned is a PlannedStep preservation constraint supplied by higher-level planning. Route-preserving optimization may move it atomically but may not drop it. The caller decides how protection behaves across a full discard/rebuild; Route Planner only enforces the flag during operations that preserve an existing affected configuration. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

Cross-route optimization is separate from BuildRoute/MaximizePartial/ExpandRoute. It receives complete effective routes for **two distinct pawns** (`receiverWorker != victimWorker`) and tests whether moving one already-assigned work item from victim to receiver, optionally releasing one unprotected receiver item, improves the complete affected pair. It performs **no victim repair and consumes no additional unassigned work**. Work Planner may later invoke other public Route Planner operations explicitly if its orchestration requires them.

## 15.1 Inputs

receiverWorker  
receiverRoute  
receiverInitialPosition  
receiverStartTime  
receiverHorizon // current-baseline preservation limit derived below\
receiverHorizonEndPosition  
  
victimWorker  
victimRoute // complete context-effective future route; contains no executing work  
victimInitialPosition  
victimStartTime  
victimHorizon // current-baseline preservation limit derived below\
victimHorizonEndPosition  

Preconditions:

- `receiverWorker != victimWorker`;
- both supplied routes are current complete effective future routes in the supplied operation context;
- both routes are already horizon-valid under their supplied fixed horizons;
- the optimizer has no MaxTimeShift authority and cannot move either horizon.

For each route at the start of a `TrySteal` call, Work Planner derives the fixed preservation limit without reading or carrying an earlier operation's `UsedTimeShift` metadata:

```text
TryStealHorizon(route) = max(
    session BaseHorizon,
    StartTime + RequiredElapsed(route, HorizonEndPosition))
```

The second term freezes the current effective route's already-authorized completion boundary; it does not grant new completion allowance. Its legality comes from the normal-session precondition for inherited routes or from the accepted operation that produced the current route, not from `TrySteal`. Receiver and victim variants must fit their independently frozen limits. Thus steal may reuse time saved by its own route rearrangement, but it cannot extend either route later than the boundary occupied by that route at call entry (or later than BaseHorizon when the entry route already fits the base). A later sequential steal call derives fresh preservation limits from the then-current effective routes; it still does not inherit operation-local shift metadata.

Work Planner constructs the immediate-steal victim set from **non-empty** other-pawn routes whose assignments were reserved/unavailable to the receiver and that contain at least one v1 removal-safe step eligible for consideration. Route Planner does not discover victim membership itself. Transient empty/no-route states and routes with no removal-safe candidate are not victims. The event/integration layer guarantees through ColonyStateContext that executing work is absent and that each operation start context is current. How those guarantees and complete-route versions are represented is outside this document.

## 15.2 Steal candidates and MustRemainAssigned

For every step X in the complete effective victim route, evaluate pure insertion and any eligible receiver-replacement variants through Eligibility.CanSteal. Pure insertion calls CanSteal with `receiverReplacedItem = null`. A receiver transition `Y -> X` is considered only when `Y.MustRemainAssigned == false` and calls CanSteal with `receiverReplacedItem = Y.WorkItem` so the complete transfer/release is validated atomically.

Steal/replacement always replans X as full execution for the receiver; victim PlannedWorkDuration metadata is not transferred. If the atomic transfer/replacement is disallowed or the receiver cannot fully execute X inside its fixed horizon, discard that variant. A victim step with MustRemainAssigned may be transferred only when CanSteal permits the resulting affected routes; the preservation flag authorizes movement, not policy bypass.

For each distinct transfer/replacement assignment state approved by CanSteal, create an isolated internal child PlanningContext under the current steal context. Speculative branches never mutate that supplied context, any ancestor context or ColonyStateContext. Each branch has context-effective receiver/victim route state associated semantically with that branch; ancestor assignments remain inherited and the child records only the information needed to represent its delta. Different receiver insertion positions for the same transfer assignment state are route-layout alternatives and do not require separate contexts. If X has MustRemainAssigned, the flag moves with X. The concrete route-delta/snapshot/overlay representation remains deferred.

## 15.3 Pure insertion into receiver

For a pure transfer, remove X from the effective victim candidate and rebuild X as a full receiver step. Test every insertion position in receiverRoute. Keep only layouts where X fully completes and receiver terminal reachability remains inside `receiverHorizon`. The victim candidate is simply the route after removal of X; there is no automatic repair or augmentation.

Under the Section 3 minimum-travel contract and its v1 removal-safe stationary-WorkItem precondition, removing X from an already-fitting victim route cannot increase required elapsed time merely because the stationary step disappeared. Therefore the resulting victim candidate remains within `victimHorizon`; no separate repair pass is required. A victim step that does not satisfy that precondition is not a valid v1 `TrySteal` candidate.

## 15.4 Replacement inside receiver

A receiver transition `Y -> X` is considered only when Y is unprotected and `CanSteal(..., stolenItem = X, receiverReplacedItem = Y)` succeeds for the complete atomic prospective state. The branch removes X from victim, releases Y into the effective unassigned pool and creates `receiverBase = receiverRoute - Y`. Then insert full X at every receiverBase position and test receiver horizon feasibility.

The released Y remains unassigned in that branch. Steal does not try to place Y into victim or any other route. If `Y.MustRemainAssigned == true`, skip the replacement variant; v1 does not implement a protected receiver-to-victim exchange.

## 15.5 Score comparison

The comparison uses the complete context-effective receiver/victim pair and the canonical `CompareRoutePairs` comparator from Section 4.1. Both route inputs are assumed current for their supplied start contexts; the optimizer does not silently refresh them for hypothetical external changes.

## 15.6 Conceptual algorithm

```text
TrySteal(receiverRoute, victimRoute):
    require receiverWorker != victimWorker
    require FitsHorizon(receiverRoute, receiverHorizon, receiverHorizonEndPosition)
    require FitsHorizon(victimRoute, victimHorizon, victimHorizonEndPosition)

    operationContext = receiverRoute.ContextId

    baseline = Current(receiverRoute, victimRoute)
    best = baseline

    for stolenStep X in victimRoute.Steps:
        for receiverReplacedItem in
                { null } + receiver steps with MustRemainAssigned == false:

            if !Eligibility.CanSteal(
                    receiverWorker, receiverRoute,
                    victimWorker, victimRoute,
                    X.WorkItem,
                    receiverReplacedItem?.WorkItem):
                continue

            branchContext = CreateChildContext(
                operationContext,
                transfer/replacement assignment delta)

            victimVariant = RemoveAndUpdateLocalMetrics(victimRoute, X)
            receiverBase = receiverReplacedItem == null
                ? receiverRoute
                : RemoveAndUpdateLocalMetrics(
                    receiverRoute,
                    receiverReplacedItem)

            for each insertionPosition of full X in receiverBase:
                receiverVariant = InsertAndUpdateLocalMetrics(
                    receiverBase,
                    X,
                    insertionPosition)

                if !FitsHorizon(
                        receiverVariant,
                        receiverHorizon,
                        receiverHorizonEndPosition):
                    continue

                candidate = {
                    receiverVariant,
                    victimVariant,
                    branchContext
                }

                if CompareRoutePairs(candidate, baseline) says Better
                   and CompareRoutePairs(candidate, best) says Better:
                    best = candidate

    release all losing internal branch contexts

    if best != baseline:
        merge best.branchContext into operationContext
        record in operationContext whatever effective-route mutation
            information is required so subsequent operations in that
            context/descendants observe best.receiverVariant and
            best.victimVariant

    return CurrentEffectiveAffectedState(operationContext)
```

The optimizer never mutates ColonyStateContext or any ancestor route state directly. If no candidate improves the affected pair, it returns with the supplied context unchanged. If an internal branch wins, Route Planner merges that branch into the supplied context and records enough context-effective state for subsequent operations to observe the winning receiver/victim configuration. The concrete representation of those changes — route delta, replacement snapshot, overlay, immutable version, change-set or another mechanism — is intentionally deferred to the dedicated context-layer design. Route Planner never performs real `ColonyStateContext` materialization itself. Inside a normal planning session, accepted steal state remains tentative until Work Planner `FinalGlobalCommit`; materialization owned by event/integration maintenance is outside this Route Planner contract.

Work Planner may then continue another sequential steal attempt. Every subsequent call must resolve the effective state after all previously accepted steals in that pass: jobs released earlier are visible in the unassigned pool, jobs transferred earlier are no longer owned by their previous route, and every affected pawn's complete effective route includes all previously accepted changes. `TrySteal` does not invoke `MaximizePartial` and does not assume that Work Planner will immediately normalize a victim route after a successful transfer; the affected pair it returns is the pair it evaluated. Any later partial normalization belongs to a later Work Planner existing-route decision boundary, the final session-wide normalization, or an explicit maintenance flow.

# 16. Deliberate v1 limitations


- Immediate steal is post-processing of a route that higher-level Work Planner has already selected. v1 does not simulate candidate-specific future steal potential while computing anchor regret, singleton alternative ranking, best-local selection or other speculative route comparisons. Doing so would multiply route candidates by victim/steal/replacement branches and is deferred unless gameplay testing demonstrates a concrete failure mode.

- Ordinary BuildRoute/ExpandRoute add only one optional job per greedy iteration.

- No X+Y lookahead: if X alone reduces Q, the planner will not add it even if X and Y together would improve the route.

- No anchor multi-start search.

- Best-local uses at most two ordinary positive-work heuristic seeds plus one additional seed for every schedulable zero-work item; zero-work seed count may be capped later only if profiling demonstrates a concrete performance problem.

- No beam search, 2-opt, swap or relocate cleanup inside ordinary route construction.

- RequiredJobs are searched exhaustively over policy-valid ordered subsets/permutations, with one maximum-feasible zero-shift terminal-partial alternative per eligible remaining item included in that exact discrete candidate space; v1 does not optimize partial duration continuously and does not perform exact optimization over arbitrary optional-job subsets.

- The v1 exhaustive multi-`RequiredJobs` guarantee is scoped to the current non-coordinated inherited-recovery use where `RequiredPrimaryCoverage = 0`. Do not extend that guarantee to coordinated contexts with order-sensitive coverage eligibility without first defining sequential-prefix versus atomic-final-state eligibility semantics.

- No assigned jobs are visible to normal route construction.

- Cross-route optimization is limited to one receiver and one victim per TrySteal call. Higher-level Work Planner may call the optimizer sequentially for multiple victim routes, with each accepted improvement becoming the baseline for the next call. Every victim is supplied as that pawn's single complete effective route in the current context; session provenance never creates a second steal target. TrySteal performs no victim repair or unassigned-work augmentation. Its removable victim items must satisfy the Section 3 v1 removal-safe stationary-WorkItem precondition.

- Higher-level sequential victim order is deliberately not optimized in v1. Because each accepted `TrySteal` changes the effective receiver/assignment state seen by later calls, a multi-victim greedy pass may produce a different final result under a different victim order. Work Planner supplies one stable deterministic order; searching victim permutations or repeating to convergence is deferred.


- Pawn-specific invalidation of a still-globally-valid assignment is deferred to the future event/integration layer. Do not add a new Route Planner mutation API or reinterpret ordinary CanUnassign/hard invalidation solely to solve that deferred event case.

- Known v1 policy limitation: generic CanUnassign preserves the last Primary of any currently Primary-containing route and does not remember whether that Primary was merely optional augmentation on an inherited-required recovery route. A rare second repair may therefore reject a valid-but-Backup-only recovered configuration. This is a safe suboptimal-repair limitation, not an ownership/correctness failure; extra provenance state is deferred unless gameplay evidence warrants it.

- No persistent cache of arbitrary job combinations is required in v1.

CompressRoute is intentionally simpler than ordinary exact RequiredJobs search: it first preserves/shrinks an existing partial when possible; otherwise each iteration tests every legal unprotected single removal together with the minimum partial shrink that would then be needed to fit. If any such variant fits, it chooses by `CompareRoutes`; otherwise it accepts the greatest strictly positive fit-time-gain removal and repeats from the unshrunk structural baseline. If no positive-gain removal exists, compression fails rather than taking a non-improving step. It never performs post-compression expansion, replacement-combination search or DFS; this loss of optimality is deliberate because compression is a rare fallback path.

| **v1 strategy —** These limitations are intentional. Advanced heuristics should be added only after gameplay testing demonstrates a concrete failure mode that the simple v1 search cannot handle acceptably. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 17. Core invariants

- Route Planner operates on abstract elementary work items rather than RimWorld-specific job semantics.

- `PlannedRoute` contains only future, not-yet-started work. Executing work is outside the route but remains assigned/reserved and unavailable to the effective unassigned pool, normal candidate snapshots and steal until the event/integration layer explicitly completes, releases or invalidates that ownership. The event/integration layer guarantees that ColonyStateContext represents the current real planning state; operation-specific `InitialPosition`/`StartTime` values are derived from that effective state; these documents do not specify how the execution layer maintains that boundary. `MaximizePartial` only attempts to increase an existing partial budget; it may return an unchanged overlong baseline and is not itself a horizon-validity proof or repair primitive.

- Every context and descendant resolves one complete effective PlannedRoute per pawn for route operations. Individual steps are not divided by planning-session provenance; concrete complete-route version storage is deferred.

- ITravelProvider, IRewardProvider and IAssignmentEligibilityProvider are long-lived dependencies. Reward values are already computed by IRewardProvider; Primary/Backup is not a numerical Reward weight. Route Planner also relies on shared planning-context infrastructure with the semantic behavior specified in this document, but the concrete context-layer API/representation is intentionally deferred.

- Negative travel duration means no path; non-negative travel duration is valid.

- Construction request timestamps/horizons may be absolute; PlannedRoute stores local durations/snapshot metrics rather than cumulative absolute per-step timestamps.

- Q(route) is the finite `TotalReward / max(TotalDuration, OnePlannerTimeQuantum)` rate; Q(empty) = 0. Provider durations are integral multiples of the canonical quantum, so only exactly zero duration uses the denominator floor. Positive Reward at zero marginal duration is a strict improvement, zero-duration regret arithmetic is finite, and IEEE infinity/NaN is never used. Generic alternative ordering is defined only by the canonical comparators in Section 4.1; higher-level callers may add policy-specific tie-breaks after an `Equivalent` result.

- Reserved terminal travel to HorizonEndPosition participates in fit checks but intentionally is excluded from route Q/WalkingDuration.

- Absolute-time feasibility is canonical: `RequiredElapsed = route.TotalDuration + terminal travel`, and a route fits only when terminal travel is reachable and `StartTime + RequiredElapsed <= Horizon`. For protected completion, `requiredShift = max(0, StartTime + RequiredElapsed - BaseHorizon)` and must satisfy the operation's completion-boundary rules plus `requiredShift <= MaxTimeShift`.

- TimeShift is operation-local and `MaxTimeShift` is always relative to the Horizon/BaseHorizon supplied to that specific Route Planner call. Positive shift is introduced only by the protected completion boundaries defined for that operation. Success may report the shift actually used when useful to the caller, but Route Planner never returns or persists a new Horizon. PlannedRoute stores neither Horizon, TimeShift nor MaxTimeShift.

- RequiredJobs complete-set/incomplete fallback is lexicographic: maximize required completion according to the documented shift rules, then apply `CompareRoutes`. A successful non-empty RequiredJobs build must represent at least one RequiredJob; zero full plus zero valid positive partial returns Failure. Work Planner may deliberately use an empty CandidateListId when it needs a pure inherited-required baseline.

- BuildRoute-created partial execution is terminal fallback within the RequiredJobs baseline search only for an incomplete RequiredJob with SupportsPartialExecution = true and uses the maximum feasible zero-shift duration after reserved terminal travel; later ordinary optional augmentation may insert full steps around that partial. `MaximizePartial` is the only normal augmentation primitive that extends an already-existing partial; positive shift may extend it only when that shift fully completes the WorkItem. ExpandRoute never extends an existing partial. A single required anchor may use partial fallback only when the result still contains at most one partial. Ordinary optional work is never partial. The global route invariant is at most one partial PlannedStep.

- PlannedWorkDuration and CompletesWorkItem are cached route-snapshot execution estimates. For a partial step, PlannedWorkDuration also serves as the current execution budget until explicit event-driven maintenance updates the snapshot. PlannedReward, TravelFromPrevious and route aggregates are cached metrics. Structural mutations maintain affected local components incrementally; any planner mutation of a partial budget atomically refreshes reward/completion/result-position, affected downstream travel, route aggregates and terminal feasibility as required by Section 2. Event-driven refresh likewise updates dependent cached values consistently and must repair/invalidate rather than silently create unsupported partial execution.

- Routes supplied to Route Planner are the complete effective routes in the supplied context; ColonyStateContext synchronization with real game state is an event/integration-layer guarantee. Normal operations do not add defensive RecalculateRoute/EvaluateRoute passes for unspecified external changes; event-driven integration maintenance owns those changes.

- BuildRoute starts from no route and never returns a successful empty route. Normal Work-Planner augmentation calls ExpandRoute only for an existing route. Empty routes are nevertheless valid transient snapshots for CompressRoute and as an affected route after TrySteal removes the final victim step; CompressRoute may return Success(empty) after legally releasing all removable work. For any operation-local empty snapshot, reward/work/walking/duration are zero and EndPosition equals that operation's InitialPosition. ExpandRoute preserves existing steps. CompressRoute may reduce an existing partial execution budget without dropping its assignment, then repeatedly evaluates unprotected policy-removable steps together with the minimum partial shrink needed after each candidate removal. It either selects the best one-removal fitting variant or greedily accepts the greatest strictly positive fit-time-gain removal and repeats. If no positive-gain removal exists, it returns `CannotFitProtectedRoute`. It returns no TimeShift/horizon metadata and never performs insertion/re-expansion, replacement or DFS search.

- Every tentative insertion from the effective unassigned pool passes CanAssign; standalone ordinary release passes CanUnassign; assigned victim-\>receiver transfer/replacement passes CanSteal. Hard invalidation of an item that no longer exists in the planning universe is not an ordinary CanUnassign operation and does not return the item to the pool. Pawn-specific invalidation of an otherwise globally valid assignment is deferred to the event/integration layer rather than modeled as another v1 Route Planner mutation. Eligibility resolves the effective context-versioned unassigned pool and each unresolved pawn's HasPlannedPrimary/PrimaryCoverageRelaxed state. In normal coordinated mode, CanAssign/CanUnassign protect the fixed RequiredPrimaryCoverage target. In relaxation mode, relaxed pawns are excluded and relevant mutations preserve the current maximum achievable coverage of the non-relaxed unresolved set (`coverageAfter >= coverageBefore`). When RequiredPrimaryCoverage is zero in a non-coordinated context, matching is vacuously skipped. CanSteal itself does not consume the stolen item from the pool and therefore does not run global matching. Any later, separately invoked assignment operation sees the resulting effective pool/context state and applies the normal CanAssign coverage rules. Context/route association is semantic here; concrete binding and storage are deferred.

- Normal BuildRoute/ExpandRoute resolve only context-owned unassigned candidate snapshots by ListId. Assigned-work redistribution occurs only through explicit cross-route optimization.

- `PrimaryCoverageRelaxed` is Work Planner-owned **context-local PlanningPawnState**, not a Route Planner result field and not ColonyStateContext pawn-global state. Once the pawn's complete effective route lacks Primary and Work Planner declares the ordinary Primary construction/augmentation stage unable to obtain one, Route Planner eligibility sees the flag through the effective context, excludes that pawn from coordinated matching, and preserves current maximum achievable coverage among the remaining non-relaxed unresolved pawns. The flag may remain true even if steal later gives that pawn a Primary. Work Planner removes the pawn at decision finalization and, for a relaxed decision, replaces the parent target with a fresh full remaining-pawn/free-work matching result.

- A MustRemainAssigned step may move atomically between affected routes but may not be dropped by that route-preserving operation. Such movement remains conditional on ordinary transfer eligibility and local Primary backing; “may move” is not a guarantee that every protected step is transferable from every route state.

- Cross-route optimization never changes fixed horizons or creates partial work. Every stolen/replacement item is full execution in the receiving route.

- Within a normal Work Planner planning session, `FinalGlobalCommit` is the Work-Planner-owned materialization boundary for session results. Route Planner works only inside the caller-supplied planning operation context, may create descendants below it, and before returning either leaves that context unchanged or merges its selected internal winner back into it. Route Planner itself never commits real `ColonyStateContext` ownership; event/integration-layer maintenance materialization remains outside this Route Planner contract.

# 18. Performance and cache strategy

- Prioritize caching expensive travel/path queries rather than arbitrary complete route combinations.

- In CanAssign, first skip coordinated matching entirely when `RequiredPrimaryCoverage == 0` and no relaxation-mode computation is active; this is the normal non-coordinated singleton/local-repair fast path. Do not substitute `PlanningPawns.Count < 2`, because a coordinated context may have one unresolved pawn with a non-zero obligation. In normal coordinated mode a reverse Primary index may skip recomputation when the fixed-target result cannot change. In relaxation mode, caching/memoization may separately reuse the current non-relaxed maximum matching, but correctness requires comparing achievable coverage before/after any mutation that can affect it. HasPlannedPrimary and PrimaryCoverageRelaxed are context-local state.

- Planned routes may cache local step metrics and route aggregates. Route Planner mutations update only affected components; this is separate from external event-driven maintenance of changed pawn/path/reward state.

- The travel provider may later use A\*, endpoint caches, room graphs and path fragments; topology-aware cache invalidation belongs there.

- Short-lived memoization within one planning event is allowed when profiling shows repeated candidates. Persistent arbitrary route-combination caching remains deferred.

# 19. Open implementation questions

- Concrete C# shapes of IWorkItem, PlannedStep/PlannedRoute snapshot metrics, BuildRoute operation-result metadata, ITravelProvider and IRewardProvider, plus IAssignmentEligibilityProvider including transfer-aware CanSteal and the context-layer mechanism that keeps each unresolved pawn's HasPlannedPrimary/PrimaryCoverageRelaxed state consistent with Work Planner decisions and accepted assignment mutations. The concrete ColonyStateContext/PlanningContext representation, complete-route version/storage model, route-snapshot binding model, mutable-vs-immutable ContextId handling, copy/merge semantics, lifetime management, pooling/reference strategy, immutable ListId registry, winning-steal victim/receiver mutation representation and the mechanism by which later sequential-steal calls observe state released/consumed by earlier accepted steals are intentionally deferred to a dedicated context-layer design; this document specifies only the required planning semantics and isolation invariants. Temporary candidate/branch route snapshots are expected and need not imply that PlanningContext itself contains a route map.

- Travel-cache invalidation strategy after topology/pathing changes.

- Concrete result metadata for reporting operation-local used/required shift where callers need it; Route Planner does not return a Horizon.

- Exact higher-level horizon/MaxTimeShift calculation is intentionally outside Route Planner: Work Planner/integration will derive the correct constraints for each call from colony state.

- Concrete partial-progress/work-time-fraction/resulting-position contract for moving work items and work families whose partial semantics are non-trivial.

- If gameplay produces RequiredJobs sets large enough that exhaustive ordered-subset/permutation search becomes a measured performance problem, revisit the search strategy based on profiling rather than imposing a v1 size contract.

- Performance limits for candidate set size before work-item clustering becomes necessary.

- Concrete integration event mapping for external pawn/path/reward changes is deferred outside Route Planner; the core contract merely assumes supplied route snapshots are current.

- Pawn-specific invalid-assignment handling for still-globally-valid WorkItems is explicitly deferred to that integration/event layer rather than another v1 Route Planner mutation contract.

- Whether a lightweight non-mutating hypothetical-context route evaluation helper is useful may be decided later from a concrete call site; v1 normal flows do not depend on routine EvaluateRoute/RecalculateRoute refreshes.

# 20. Settled v1 decisions / review guardrails

The following are deliberate v1 decisions and should not be reopened in routine review without new evidence, a concrete failure mode or an implementation contradiction.

- Route Planner does not expose BuildPolicyRoute and does not interpret Primary/Backup. Caller candidate pools and eligibility enforce policy. `MaximizePartial` is a separate public normalization primitive and is never an implicit ExpandRoute phase. It never shrinks the existing partial budget; partial reduction belongs to CompressRoute.

- Public `BuildRoute` and public `ExpandRoute` share the Section 11 private ordinary optional-augmentation helper. Invoking that helper is an internal phase of the enclosing public operation, not a nested public `ExpandRoute` call and not a new Work-Planner normalization boundary. An omitted/incomplete original RequiredJob likewise cannot become a newly completed ordinary optional insertion under the same BuildRoute call-local limit without contradicting the exact required-stage maximum-completion result; do not add provenance/filtering machinery for that false-positive scenario.

- `MaximizePartial` normalization guardrail: the operation only attempts to increase an existing partial budget and never shrinks/removes work. An already-overlong baseline may therefore be passed through and returned unchanged when no legal increase exists. Do not interpret `Success` as a horizon-validity proof and do not make `MaximizePartial` silently repair the baseline; admission/maintenance/compression remains caller-owned.

- `BuildRoute Success` always means a non-empty route. With an empty RequiredJobs set at least one ordinary candidate must be scheduled; with a non-empty RequiredJobs set at least one RequiredJob must be represented fully or by a valid positive-duration partial. Otherwise BuildRoute returns `Failure` and leaves the supplied operation context unchanged.

- The non-empty-success rule is specific to BuildRoute. Existing-route operations may still produce transient empty snapshots: CompressRoute may Success(empty), and TrySteal may leave the victim route empty after transferring its final step. Such a snapshot has zero reward/work/walking/duration and EndPosition equal to the operation InitialPosition. Route Planner does not persist it or reinterpret it as Idle; Work Planner owns that orchestration decision.

- `RequiredJobs` does not imply `MustRemainAssigned`. The same generic mechanism is used for anchor targeting and inherited-recovery constraints. Current Work Planner marks an anchor step protected only when an anchor-targeted candidate becomes the selected route; inherited rebuild protection is likewise higher-level bookkeeping on surviving protected work. Neither case requires Route Planner to infer provenance from RequiredJobs.

- Existing route snapshots are assumed current. Do not add routine defensive RecalculateRoute/EvaluateRoute calls for unspecified future game-state changes; integration consistency is event-driven and may update cached PlannedWorkDuration/CompletesWorkItem plus dependent metrics when explicitly handled events occur.

- Local structural edits update local composable snapshot metrics incrementally rather than forcing full-route recomputation.

- Persistent PlannedRoute contains no TimeShift/MaxTimeShift and no required cumulative absolute per-step timestamps.

- Terminal travel to HorizonEndPosition is deliberately excluded from Q/WalkingDuration and used only for feasibility.

- Positive shift is operation-local authority to cross an allowed completion boundary within the Horizon/BaseHorizon and MaxTimeShift supplied to that call. Route Planner does not infer the constraints of later public calls from an earlier result; the caller independently supplies the correct constraints each time.

- BuildRoute creates partial work only for an incomplete RequiredJob with SupportsPartialExecution = true, using the one maximum feasible zero-shift duration after reserved terminal travel; ordinary best-local construction has neither partial fallback nor construction TimeShift. The maximum partial duration is a deliberate maximum-progress policy rather than Q optimization over shorter durations. `MaximizePartial` owns later extension of an existing partial; ExpandRoute may insert full work around the normalized partial without changing its budget.

- Primary/Backup is not numerical reward weighting; IRewardProvider is opaque and owns partial-progress reward semantics.

- v1 planner/simulator assumes elementary WorkItems; clustering, packages, multi-interaction-cell choice and other RimWorld-specific decomposition remain deferred.

- Review guardrail for removal-safe route operations: v1 `CompressRoute` and `TrySteal` accept only WorkItems satisfying the Section 3 removal-safe stationary precondition. Under that precondition plus the fixed-topology minimum-travel contract, deleting an intermediate step does not require inventing a new “unreachable bridge” failure. Movement-providing, topology-changing or otherwise removal-sensitive WorkItems are deferred and must not be silently passed into these operations.

- Failed `CompressRoute` is transactionally invisible to its caller: all accepted tentative removals live only in the caller-created child operation context, and `CannotFitProtectedRoute` causes that context/subtree to be discarded so the parent state remains unchanged.

- Non-coordinated singleton/local-repair contexts may still contain their one affected PlanningPawn; Work Planner disables coordinated coverage by supplying `RequiredPrimaryCoverage = 0`. CanAssign may fast-path that case, but must not infer the absence of coverage solely from the number of PlanningPawns.

- Tentative ownership and coordinated planning state are infrastructure-owned and addressed through the effective planning context associated with each route alternative. Work Planner owns the current RootSessionContext and creates independently compared Route Planner answers as sibling operation contexts from the same unchanged common decision baseline supplied by Work Planner. Route Planner may create descendants only below the context it receives, destroys/releases losing internal branches, and merges its selected internal state back into that supplied context before return. The effective unassigned pool and unresolved-pawn HasPlannedPrimary/PrimaryCoverageRelaxed facts are context-versioned state; Candidate/Required collections are immutable call snapshots and are not authoritative ownership state. Temporary route snapshots are expected during candidate/branch evaluation, but their concrete storage/binding is intentionally deferred. Do not turn PrimaryCoverageRelaxed into a global or persistent pawn flag/set: it exists only on unresolved PlanningPawnState in the active context. Speculative child branches never mutate parent contexts or parent route snapshots. Do not reintroduce stack-local surviving overlays, raw long-lived list ownership inside Route Planner, global pawn IsPlanning, or persistent C_committed/current-worker-credit fields.

- Primary-coverage relaxation is represented only by `PlanningPawnState.PrimaryCoverageRelaxed` inside the active candidate/context. It is not a BuildRoute/ExpandRoute/TrySteal result bit and is never persisted after that pawn decision. During relaxation, do not keep protecting a stale fixed target; preserve the current non-relaxed maximum achievable coverage, then let Work Planner fully recompute the parent target when the relaxed pawn is finalized.

- `HasPlannedPrimary` is a context-local cache over the pawn's complete effective route. Any Primary anywhere in that route counts equally; do not introduce planning-session provenance or a route-boundary exception.

- Accepted route/steal changes remain context-effective tentative state until the owning higher-level transaction materializes them. In a normal planning session that boundary is `FinalGlobalCommit`; event/integration-owned maintenance materialization is deferred separately. Route Planner must not directly edit ColonyStateContext or ancestor context state; the concrete full-state/delta/snapshot/overlay mechanism is deferred.

- Known v1 limitation: repeated repair of an inherited-required recovery route may over-preserve an optionally added last Primary because generic eligibility does not store route provenance. This is intentionally accepted as a rare safe/suboptimal case. Review guardrail: MustRemainAssigned allows a protected step to move atomically only when ordinary CanSteal/local-backing rules permit the resulting state; wording that a selected/protected anchor “may move” is intentionally conditional and does not promise universal transferability.

- Immediate-steal victim membership is a Work Planner assignment-visibility decision: Route Planner only receives a specific non-empty other-pawn victim with at least one removal-safe candidate step. `receiverWorker == victimWorker` is invalid. ColonyStateContext/ancestor effective routes can be victims when their assignments were reserved/unavailable to the receiver; unresolved speculative sibling alternatives and transient empty/no-route states are not victims. Work Planner derives each call's receiver/victim preservation horizon from the greater of BaseHorizon and that route's current absolute required end, with no cap inferred from unused MaxTimeShift. This freezes an already-authorized baseline boundary without carrying earlier operation result metadata or granting new shift.

- Execution-boundary guardrail: do not reintroduce an executing/locked prefix inside `PlannedRoute`. Executing work is outside the route but remains assigned/reserved and absent from the effective unassigned pool, normal CandidateLists and steal victims until the event/integration layer explicitly completes, releases or invalidates that ownership. Route Planner receives only the still-planned route plus its already-correct start context; the future event/execution integration design owns the concrete reservation/transition mechanics.

- Sequential-steal effective-state guardrail: every later `TrySteal` call in one pass must observe all earlier accepted assignment releases/consumptions and effective-route mutations through the caller context. Subsequent operations always see each pawn's complete effective route. Do not prescribe a concrete CandidateList refresh, full-state/delta, snapshot, overlay or merge implementation here; only the effective-state result is part of the Route Planner contract until the dedicated context-layer design is written.

- TrySteal is post-selection optimization. Do not invoke it speculatively for every route candidate merely to fold future steal potential into higher-level candidate scoring unless profiling/gameplay evidence justifies that larger search.

- Comparison-policy guardrail: generic route and affected-pair ordering is defined only by `CompareRoutes` / `CompareRoutePairs` in Section 4.1. Algorithm sections should reference those comparators after their operation-specific higher-priority criteria instead of duplicating Q/walking/tie-break sequences.

- Steal-normalization guardrail: `TrySteal` evaluates and returns the affected route pair without implicit partial normalization. Do not reintroduce a requirement that every accepted steal immediately triggers `MaximizePartial`; later normalization belongs to the next Work Planner existing-route decision boundary, the final session pass, or explicit maintenance.
