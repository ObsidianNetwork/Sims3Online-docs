# Routing

Routing has two jobs in the simulation: decide where a Sim should go and move that Sim through the game's local navigation system. The online mirror adds a third concern: communicate enough intent and presentation state for another Sims instance to show the same movement without assuming its native route internals are identical.

This page consolidates the interaction-level routing contract, GoHere, RouteAway and mounted RideHere behavior, route recording, position/speed synchronization, and the opt-in external route-planning seam. Current entry points are `SimsOnlineServerHostMod\RoutingInteractionFilter.cs`, `SimsOnlineServerHostMod\RouteRecorder.cs`, `SimsOnlineServerHostMod\Presentation\Walks.cs`, `DedicatedServer\HostBridge\RoutePlanRequestLine.cs`, `DedicatedServer\Authority\RoutePlanHop.cs`, and the interaction/snapshot records under `SimsOnline.Protocol\`.

## Three different routing layers

| Layer | Responsibility | Current representation |
|---|---|---|
| Interaction intent | Says that a Sim chose a destination or routing interaction. | `InteractionQueued` with interaction key, target object id, and/or target position. |
| Native route plan and execution | Chooses a reachable goal/slot, computes route actions, follows portals/paths, and reports completion or failure. | Game runtime route object and its `RoutePlanResult`; these are local to a Sims instance. |
| Multiplayer presentation | Makes the remote copy move, corrects its final position/facing, and handles game speed. | `SimAnimation` presentation lines plus snapshot `Position`; route streaming is opt-in. |

A route object's handle, path nodes, and live action state are not public cross-process identifiers. Clients should not assume that two native planners choose the same chair slot, doorway, or path merely because they receive the same interaction name.

## Interaction-level route contract

`InteractionQueued` in `SimsOnline.Protocol\ServerEvent.cs` carries:

```csharp
InteractionQueued(
    ulong ActorObjectId,
    ulong InteractionId,
    string InteractionKey,
    ulong? TargetObjectId,
    Vector3Dto? TargetPosition,
    IReadOnlyDictionary<string, string>? Parameters = null)
```

The target's object id can be used to resolve a local object; the position is a portable destination hint. For a recognized route key, the host mod captures `GetTargetPosition()` only when it is a finite in-world point. For an ordinary targeted interaction, the host may instead send the target object id and its readable position so the client can walk toward the object without replaying the unknown activity itself.

The dedicated-server `InteractionReplayPolicyResolver` uses these fallbacks:

- Recognized routing key and a target: `ReplayAsRoute`.
- Recognized terminal with a target: `ReplayExact` (unless a posture transition is selected first).
- Unknown key with target: route-to-target degradation, not arbitrary local interaction creation.
- Unknown key with no target: `SlideOnly`; the position/presentation stream is the only available visual update.

The script mod's route apply path converts the target into a local `Terrain.GoHere` interaction and queues it with `AddNext`. If the event supplied a position, it uses that point; if only an object id was supplied, it resolves the object's local `Position`. This is why a non-null target does not prove that the local game will find an equivalent object.

### Current route-key allowlist

`RoutingInteractionFilter` is a fail-closed classifier for these namespaced families:

| Prefix | Meaning in this mirror |
|---|---|
| `Sims3.Gameplay.Core.GoHome` | Route toward home. |
| `Sims3.Gameplay.Core.VisitCommunityLot` | Travel toward a community lot. |
| `Sims3.Gameplay.Objects.Vehicles.FollowerMeetUpAtPoint` | Group-route follower approach. |
| `Sims3.Gameplay.Objects.Vehicles.LeaderGoHereWith` | Group-route leader approach. |
| `Sims3.Gameplay.Core.Terrain+GoHere` | Fixed-point intra-lot GoHere. |

Definition keys include nested type names such as `Sims3.Gameplay.Core.Terrain+GoHere+Definition`. The classifier matches a namespace-qualified type boundary; an unrelated class with the same leaf name is not accepted. The explicit `Terrain.GoHere` friendly key is also recognized by the application path.

The list intentionally does not include `Terrain+RouteAway` or `Terrain+RideHere`. Those are not ordinary fixed-destination walks, as described below.

## GoHere and destination coordinates

A GoHere expresses a fixed destination. Same-lot and other-lot variants can choose different engine options, but the wire-level mirror still needs a concrete position and then lets the local game plan a route. The client does not receive a guarantee of identical path geometry.

Y is part of the destination. A destination with correct X/Z but a default Y of zero can be below the lot's walk surface. The server mod calls `SnapDestinationToTerrain` before queueing GoHere: it compares the requested Y with `World.GetTerrainHeight(x, z)` and substitutes terrain height when the difference exceeds `0.5`. This is applied to replayed positions and resolved target positions. It is not a general coordinate conversion; it only repairs an implausible terrain-height mismatch.

The route planner can still fail after GoHere is queued. A successful interaction insertion means the queue accepted the request; it does not mean `Route.Plan()` found a path or that the Sim reached the destination.

### RouteAway is not GoHere

`RouteAway` asks the engine to wander around the actor; it does not expose a stable fixed destination at the same time that an ordinary routing interaction does. Replaying it as a GoHere to its initial position is a no-op, while asking each client to wander independently can choose different endpoints. Keep it outside the fixed-route allowlist. Position/presentation mirroring is safer than pretending its random wander is an explicit route.

### RideHere is not a foot route

`RideHere` is a mounted-terrain interaction. Its route can move a horse/mount while coordinating a rider and mounted posture. Converting that to `Terrain.GoHere` on an ordinary puppet would make the puppet walk on foot and omit the mount synchronization. It is therefore not interchangeable with GoHere, even if an endpoint is available. The current routing filter does not include it.

## Native planning: goals, results, then movement

The Sims route system selects a start and one of the eligible destination candidates, applies route options and object/slot constraints, plans path segments, and executes route actions. Portals and vehicles can divide one trip into several legs. A route goal commonly includes more than XYZ: a destination tag, optional object/slot identity, and end facing can matter. The path itself is a local plan and can differ between machines.

The managed API exposes `Route`, `RoutePlanResult`, `RoutePlanResultType`, `RouteOption`, `RouteOption2`, `RouteStopEventType`, and `FailureExplanation`. The essential diagnostic distinction is:

| Stage | What failure means | Example evidence |
|---|---|---|
| Goal/start planning | A usable route was not produced. | `RoutePlanResult.Succeeded()` is false; result type may identify missing destination or no path. |
| Route execution | A plan existed, but its movement/action failed or was interrupted. | Stop event such as `Failure`, `DestinationObstructed`, or `EncounteredObstacle`; `FailureExplanation` may identify portal, vehicle, or replan failure. |
| Posture transition | A queued activity could not reach its required posture before route planning. | No route result exists; inspect `Posture`/queue and transition state instead. |
| Mirror divergence | Both Sims reach a destination but their local paths/positions differ. | Compare end position, goal identity, route completion, and presentation timing. |

Representative `RoutePlanResultType` values include `Succeeded` (1), generic `Failed` (2), `NoDestinationSpecified` (10), type 16 (no path), and `DestinationPortalLocked` (18). Generic `Failed` is not enough to diagnose the cause; inspect whether planning produced a result and whether route execution later stopped. Do not treat a queued interaction failure, planner failure, and interrupted route as one error category.

`RouteOption` and `RouteOption2` include families for vehicles/strollers, mounted or animal movement, water, portals, obstruction checks, and route-shaping choices. Their values affect local planning; a mirror should not serialize the entire native options block as if it were a stable public API.

## Route presentation stream

The server mod can record the route it actually follows through `RouteRecorder`, a `ScriptCore.Route` subclass installed at the engine's route-constructor seam. It records route start, walk-style change, stop, and end. The presentation handlers in `SimsOnlineServerHostMod\Presentation\Module.cs` register route-line consumers, and `Presentation\Walks.cs` drives a local route for the client copy and holds position/animation lines while that walk is active.

This stream is separate from the public `InteractionQueued` event. It is carried as presentation data (`ServerEvent.SimAnimation`), not as a new public route command. The primary line families and fields are:

| Line | Current field shape | Purpose |
|---|---|---|
| `Rs` | `Rs|sim|goalX|goalY|goalZ|tag|destSlot|destObj|waypoints|startMs|style|age|gender|species|flags|route` | Start a recorded walk using the server's goal, optional waypoints, and walk style. |
| `Ru` | `Ru|sim|style|age|gender|species|route` | Change the walk style for a current or next leg. |
| `Rh` | `Rh|sim|x|y|z|route|stopMs|mode|moved|leftBefore|leftAfter|sinceMs|sinceTicks` | Report a stop and when in the walk it occurred. |
| `Rx` | `Rx|sim|x|y|z|route|endMs|fx|fz|by|stopMode|stopAccepted|stopMs` | End the route at the observed position/facing and record why it ended. |
| `Pz` | `Pz|sim|x|y|z` | Position/facing correction feed for a streamed Sim. |

The exact fields are presentation bridge records, not general client API types. A route is sent only when its goal is meaningfully different from the Sim's start; `RouteRecorder` currently uses a `0.3` world-unit minimum distance. A route through a door can yield multiple route legs against one overall goal. A start for the same goal can continue the existing client walk rather than starting an unrelated one.

The stream is off by default. `DedicatedServer\Program.cs` enables it only when the presentation stream is active and `SIMSONLINE_ROUTE_STREAM=1` is set. In `RouteRecorder`, route events are only published when the recorder/ring and route stream are enabled. The client stops a route at the recorded end, releases lines held behind the walk, and falls back to local position presentation when route planning fails.

Position stream and snapshot positions serve different purposes:

- `SimSnapshot.Position` is part of `SimulationSnapshot`; it is a world-state observation used by snapshot/mirror processing.
- `Pz` presentation lines give the active presentation path a faster position target. `Presentation\Walks.cs` tracks position changes at `0.05` units and applies a correction only when the copy is more than the default `0.25` deadband away and the relevant rest/posture rules allow it.
- While a streamed Sim is walking or has a recent presentation position, the slower snapshot position mirror is held off to avoid snapping the copy backward during an active route.

Position correction is not a substitute for posture alignment. A Sim on a chair or bed is positioned by the object's animation/seat machinery; snapping the Sim's origin to a world coordinate can visibly move it out of the furniture.

## Game speed synchronization

The world snapshot includes a `Speed`, and `ClientCommand.SetGameSpeed` accepts one of the `GameSpeed` values `Pause`, `Normal`, `Speed2`, `Speed3`, or `Ultra`. A client-role UI speed change can be forwarded as `SpeedIntentCaptured`; the server adopts the authoritative value and broadcasts `GameSpeedChanged`. The server also includes known speed state for joining players.

Speed is simulation clock control, not a route parameter. A paused simulator may accept a GoHere command but not advance the route or posture transition. Do not use a fixed wall-clock delay as proof that a paused route completed.

## Optional alternate planner seam

`RouteRecorder` also wraps `Plan`, `PlanFromPoint`, `Replan`, and `ReplanFromPoint`. The alternate Rust seam is opt-in with `SIMSONLINE_RUST_ROUTE=1`; it asks the controller for finite XYZ waypoints and inserts a custom Sim path only when the route has a Sim follower, a point goal, no existing path, and a matching plan.

The internal bridge request has this form:

```text
EVENT|RoutePlanRequest|sim|sx|sy|sz|gx|gy|gz|corr
```

Coordinates are invariant-culture floating-point metres. The `corr` suffix correlates a waiting request/response. The response is a `SACS_RECORDER rust-plan` with `x,y,z,x,y,z,...` points, or `rust-plan-none`. `RoutePlanRequest.TryParse` rejects missing or non-finite coordinates; a plan must contain at least two finite XYZ points. `RustRoutePlan.GoalTolerance` is `0.10` in X/Z: the last point must end near the requested goal. A plan for a different goal remains unused rather than being applied to the wrong route.

Waiting is bounded by the `route.rust.wait.seconds` tunable (default `0.75`, clamped to `0.05`–`5.0` seconds). Calls made on a non-yielding thread or inside the bridge pump do not wait. A declined request, timeout, invalid/mismatched plan, unsupported goal, or failed custom-path insertion falls back to the native planner. Exceptions caused by a Sim reset or thread abort are not swallowed as ordinary planner fallback.

The current `KernelPendingRoutePlanSource` returns no plan. Thus enabling the seam exercises request/decline/fallback behavior; it does not replace the game planner with a functioning general-purpose Rust route engine. The default route path remains native.

## Failure modes and investigation

| Symptom | Likely boundary | Check |
|---|---|---|
| GoHere queues, then ends almost immediately | Planner rejected the goal, often due to an invalid target height or blocked destination. | Read the interaction completion/failure and route result; compare Y with terrain height. |
| Sim never starts moving while the world is paused | Route/posture coroutines require simulation ticks. | Check authoritative `GameSpeed`; distinguish queue acceptance from route completion. |
| Copy teleports or jumps backward during a walk | Snapshot positions arrived apart from presentation/route timing. | Check whether `SIMSONLINE_ROUTE_STREAM` is active and whether snapshot position application was held. |
| Sim walks to the wrong chair | Goal object is right but destination slot/seat was not preserved. | Inspect route goal slot and captured `seatArea`/`seatContainmentSlot`. |
| A route appears to succeed but the client has a different endpoint | Independent local goal/path selection or missing route end correction. | Compare goal tag/slot/object and `Rx`/`Pz` final positions, not just the key. |
| `RouteAway` looks like a no-op or different wander | It has no portable fixed destination. | Keep it out of `RoutingInteractionFilter`; use position presentation. |
| `RideHere` becomes a walking animation | Mounted route was flattened into GoHere. | Keep mount/rider posture and route as a separate feature family. |
| Rust route requests always use native planning | No valid plan source replied, or eligibility/goal checks failed. | Inspect `RustRoutePlan.Report()` counters and `rust-plan-none`/fallback reason. |

## Verification

Current tests that cover the route boundary include:

- `DedicatedServer.Tests\Program.cs`: `RoutingInteractionFilter.AllowsRoutingDefinitions`, `RoutingInteractionFilter.FailsClosedForNonRouting`, `RoutingInteractionFilter.DoesNotCollideWithSameLeafNameTypes`, `UnclaimedRoutingEmitFilter.EmitsTier1UnclaimedRoute`, and `RoutingInteraction.QueuedWithTargetReplaysAsGoHerePreservingKey`.
- `DedicatedServer.Tests\RustRouteSeamTests.cs`: `RustRoute.OffByDefaultIsNativeAndUncounted`, `RustRoute.PlanIsTakenOnceAndOnlyForItsGoal`, and `RustRoute.OfferRefusesBrokenPlans`.
- `DedicatedServer.Tests\RustRouteWaitTests.cs`: request parsing, correlated response, decline, timeout, non-yielding context, and late-offer behavior.
- `DedicatedServer.Tests\Program.cs`: `AuthoritativeSession.TurnsRouteStreamOnWithTheRing`.

These tests verify classifiers, wire shapes, and fallback rules. A live route to a specific object can still depend on the loaded world, slot occupancy, simulation speed, and each instance's native planner.

## Related pages

- [Postures and transitions](postures-and-transitions.md)
- [Terminals](terminals.md)
- [Protocol](../protocol.md)
- [Simulation model](../simulation.md)