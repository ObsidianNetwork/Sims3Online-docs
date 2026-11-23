# Interactions and lifecycle

An interaction is the server game's unit of attempted work; a protocol command is a request to create or alter that work, and a client-side route or animation is a presentation of authoritative state. The public implementation connects these layers through `ClientCommand.QueueInteraction`, `ServerEvent.InteractionQueued` and lifecycle events, and the script-mod bridge. This page documents those boundaries, the mirror batching rules, the task-scheduler hooks used by the mod, and the information that can be observed without assuming engine internals.

## The public lifecycle in one view

```mermaid
sequenceDiagram
    participant UI as Client game / UI
    participant C as SimsOnline.Client
    participant S as Dedicated server
    participant M as Server host mod
    participant G as Authoritative game
    UI->>C: user pick intent
    C->>S: QueueInteraction(commandId, playerId, request)
    S->>S: joined-player and actor-ownership checks
    S->>M: QUEUE_INTERACTION bridge command
    M->>G: rebuild pick; validate; enqueue
    G-->>M: start / completion / failure callbacks
    M-->>S: InteractionQueued / Started / Completed / Failed
    S-->>C: command result and scoped events
    C->>M: replay supported events on mirror
```

This describes the observable integration, not a promise that every accepted command starts. Queue acceptance and execution have separate outcomes: an accepted command can later fail, and cancellation is not equivalent to completion.

## Data model and wire records

`SimsOnline.Protocol/ClientCommand.cs` defines command variants. For interactions, the important record is `QueueInteraction(Guid CommandId, int PlayerId, InteractionRequest Interaction)`. `InteractionRequest` contains:

| Field | Type | Meaning |
|---|---|---|
| `ActorObjectId` | `ulong` | Game object ID for the Sim receiving the command. |
| `InteractionKey` | `string` | Definition identity; the host resolves it against the actor/target's offered interaction list. |
| `TargetObjectId` | `ulong?` | Target identity when the action has one; null can represent terrain. |
| `TargetPosition` | `Vector3Dto?` | Destination or hit position when required by the command. |
| `Parameters` | `IReadOnlyDictionary<string,string>` | Family-specific hints or the `ui.*` fields of a captured player pick. |

A player-pick request is identified by the presence of `ui.name` in `Parameters` (`UserPick.IsUserPick`). The related keys are `ui.path`, `ui.hit`, and `ui.picked`; they carry the menu choice, hit description, and selected objects. The wire shape is additive JSON, for example:

```json
{
  "$type": "queue_interaction",
  "commandId": "f789a654-7c84-4bc4-a133-c403c15d0a6c",
  "playerId": 12,
  "interaction": {
    "actorObjectId": 7001,
    "interactionKey": "Sims3.Gameplay.InteractionsShared.Sit+Definition",
    "targetObjectId": 8002,
    "targetPosition": null,
    "parameters": {
      "ui.name": "Sit",
      "ui.path": "",
      "ui.hit": "0:0:0",
      "ui.picked": "-"
    }
  }
}
```

The host-mod side receives a `QUEUE_INTERACTION` bridge line. `DedicatedServer/HostBridge/BridgeLineCodec.cs` encodes user picks with the click name/path, hit and picked-item data in addition to the ordinary actor/target fields. The mod reconstructs the candidate against the server's own offered interactions; it does not trust a client-supplied arbitrary queue object.

### Events and identifiers

`SimsOnline.Protocol/ServerEvent.cs` defines distinct records rather than one generic “interaction result”:

| Event | Fields that identify the activity | Interpretation |
|---|---|---|
| `InteractionQueued` | `ActorObjectId`, `InteractionId`, `InteractionKey`, optional target ID/position, optional `Parameters` | The host observed/accepted an instance for queueing; target-bearing events can support mirroring. |
| `InteractionStarted` | actor and interaction IDs | Start callback for the specific instance. |
| `InteractionCompleted` | actor and interaction IDs | Completion callback; clients may end active-order position suppression. |
| `InteractionFailed` | actor and interaction IDs plus `Reason` | Callback failure with diagnostic context; not the same as a pre-queue rejection. |
| `InteractionReplayQueued` | an `InteractionReplayBundle` | Richer replay capture, with its own parser and policy path. |
| `CommandAccepted` / `CommandRejected` | command ID, and rejection reason where applicable | Result for the request, not lifecycle completion. |

A target-bearing event is serialized in the public JSON shape below. `parameters` is optional: old emitters can omit it, and consumers treat null as empty.

```json
{
  "$type": "interaction_queued",
  "actorObjectId": 7001,
  "interactionId": 4495,
  "interactionKey": "Terrain.GoHere",
  "targetObjectId": null,
  "targetPosition": { "x": 11.0, "y": 0.0, "z": 22.0 },
  "parameters": {}
}
```

`InteractionQueued.Parameters` carries semantic values when emitters have them; examples produced by `SimsOnlineServerHostMod/ServerHostMain.cs` include `bookId`, `progressNormalized`, `seatArea`, `seatContainmentSlot`, `seatRoutingSlot`, and `seatIkSuffix`. These are strings so the public envelope can carry family-specific data without changing the protocol type for every interaction family. Consumers must still validate or ignore unrecognized keys.

The mod's event callbacks in `SimsOnlineServerHostMod/ServerHostMain.cs` send `EVENT|InteractionStarted`, `EVENT|InteractionCompleted`, and `EVENT|InteractionFailed`. The latter includes key, speed, role and best-effort position in its diagnostic text. The interaction ID is read from the actor's current state, with a tracked-ID fallback for starts and completions. Treat actor plus interaction ID as the correlation pair; an empty key at failure time is possible because diagnostic state is sampled after the callback boundary.

## From pick to authoritative queue

For a captured player pick, `SimsOnlineServerHostMod/ServerHostMain.cs` performs these steps in `HandleQueueUserInteraction`:

1. Resolve the actor by its server object ID; fail with `Sim not found` if unavailable.
2. Resolve a target ID, or use `Terrain.Singleton` for a terrain pick; fail with `Target not found` if resolution fails.
3. Enumerate the relevant menu entries and match definition type, display name and path; a single matching definition type is accepted as an unambiguous fallback.
4. Rebuild the hit and selected-object parameters from the forwarded fields. A malformed or unresolved selected object is rejected rather than silently substituted.
5. Create the instance at `UserDirected` priority with user-directed flags and callbacks.
6. Queue it, or report a rejection reason and remember refused menu entries for presentation.

The server is the authority for actor ownership. `DedicatedServer/Authority/AuthoritativeSession.cs` rejects an unjoined player and rejects an interaction on a Sim claimed by another player before calling the host. A `QueueInteraction` returning accepted means this boundary accepted the request; it does not promise that a route, posture transition, or action body will finish.

## Coalescing interaction events before replay

`SimsOnline.Client/MirrorCoordinator.cs` accumulates adjacent `InteractionQueued` records and flushes them through `DedicatedServer/Mirroring/InteractionQueuedCoalescer.cs`. Events remain ordered except where the following explicit reductions apply:

| Rule | Current behavior | Example |
|---|---|---|
| Same actor, same interaction key | A later-or-equal `InteractionId` replaces an earlier instance of that key. | Two GoHere updates retain the latest endpoint. |
| Same actor, two route-replay keys | The newer route intent supersedes earlier route churn. | A new route replaces an earlier route leg. |
| Different terminal keys | Keep both in arrival order. | Sit followed by ReadBook remains a chain. |
| Duplicate tail | Skip an event matching the last result's actor, key and interaction ID. | A repeated delivery is not dispatched twice. |
| Social chain reduction | If `SocializeWith` exists for an actor, drop that actor's `SocialInteractionA` / `SocialInteractionB` approach events. | The terminal conversation event is enough to dispatch the mirror action. |
| Book-chain reduction | If a ReadBook terminal exists, drop `ReadBookChooser` entries for that actor. `GetBook` and Sit are not stripped by this pass. | Preserve GetBook, Sit, ReadBook while removing chooser noise. |

The algorithm is per actor and does not merge parameter maps. A batch that leaves two or more events may use `MirrorCoordinator`'s batch dispatch path; a single remaining event is dispatched individually. The separate replay-bundle path has its own policy resolver and should not be inferred from these public-event rules.

## Mirroring and active orders

A snapshot position and an interaction destination mean different things. `MirrorAutonomyController` mirrors positions from snapshots but deliberately does not replay a snapshot's current `GoHere` key: a snapshot gives current position, not the original destination. The event path can replay when `InteractionQueued` carries a target position or resolvable target object; `EventToBridgeCommandTranslator` filters out never-replay keys and targetless interactions.

For a client-owned Sim performing a directed route, `SimsOnline.Client/Program.cs` uses `ActiveOrderTracker` to temporarily exclude that actor from authoritative position application while its local route runs. The tracker:

- is keyed by actor, not command ID;
- starts on confirmed/replayable order handling;
- clears on `InteractionCompleted` or `InteractionFailed`;
- expires entries after 60 seconds as a backstop;
- does not disable autonomy itself and does not change idle-Sim mirroring.

After completion/failure, `OrderCompletionReassert` can reapply the latest authoritative position so the last update omitted during the strip window is not lost. The independent pending-intent tracker is shorter-lived: it exists to match a request to its authoritative echo and is consumed at confirmation, so it cannot represent the whole walk.

## Scheduler boundary used by the mod

The mod registers recurring or deferred work with the game scheduler instead of running every operation directly in the bridge reader. `ServerHostMain` starts its bridge pump with `Simulator.AddObject(new OneShotFunctionTask(...))`; periodic work is registered through `EveryUpdateTask`. Route-related code such as `SimsOnlineServerHostMod/RouteRecorder.cs` checks `Simulator.CurrentTask`, tests whether yielding is allowed, and yields with `Simulator.Sleep(0u)` before continuing its wait loop. Some presentation and interaction helpers also submit `OneShotFunctionTask` work to the simulator.

This separation matters for reviewers: a bridge acknowledgement proves the handler accepted a command, not that scheduled in-game work completed on the same thread or tick. Do not copy timing guarantees from wall-clock waits into simulation behavior. The current checked-in project documents the scheduler call sites and cooperative-yield contract; it does not include a public, independently testable specification of the game's private scheduler internals.

## Failure stages and diagnosis

| Stage | Example evidence | Typical interpretation |
|---|---|---|
| Connection or role gate | `command_rejected` before a host call | Wrong join state, connection identity or role. |
| Session ownership | Rejection says the Sim is claimed by another player | Actor ownership failed; no queue attempt occurred. |
| Actor/target lookup | `Sim not found`, `Target not found`, or unresolved picked object | Server world identity differs from or lacks the client's reference. |
| Pick lookup | `No matching interaction` | The actor/target currently offers no definition matching the forwarded menu selection. |
| Queue insertion | `InteractionQueue.Add refused ...` | The engine refused insertion; when available, the server's greyed-out tooltip is appended. |
| Accepted then not run | `InteractionFailed` or `PickFailed` | A later callback or queue-stage failure; inspect reason and lifecycle IDs. |
| Mirror candidate missing target | No translated `ApplyInteractionQueue` | The public event is not target-bearing or is excluded from replay. |
| Route still active after stale event loss | Position remains stripped until TTL | Completion/failure event may be missing; active order expires after 60 seconds. |

A successful replay test should verify more than queue acceptance: compare `interaction_queued`, start/terminal events, target, final authoritative snapshot, mirror bridge result, and visible client outcome.

## Tests and source map

The interaction plumbing has focused tests in `DedicatedServer.Tests/Program.cs`:

- `InteractionQueuedCoalescer.LatestPerActorWins`
- `InteractionQueuedCoalescer.PreservesTerminalChain`
- `InteractionQueuedCoalescer.StripsBookPrecursorsWhenReadBookPresent`
- `InteractionQueuedCoalescer.StripsSocialRoutesWhenSocializeWithPresent`
- `ActiveOrderTracker.MarkIsActiveClear`, `ActiveOrderTracker.ExpiresAfterTtl`, and `ActiveOrderTracker.ReMarkResetsTtl`
- `MirrorEngine.EventPathStillReplaysGoHereWithRealTarget` and `MirrorEngine.SnapshotPathDoesNotReplayGoHereInteraction`
- `OutboundListenerTests` includes `ClientListenerServer.SendsAFailedPickToItsPlayerOnly`.

Core references: `SimsOnline.Protocol/ClientCommand.cs`, `SimsOnline.Protocol/ServerEvent.cs`, `DedicatedServer/Authority/AuthoritativeSession.cs`, `DedicatedServer/Mirroring/EventToBridgeCommandTranslator.cs`, `DedicatedServer/Mirroring/InteractionQueuedCoalescer.cs`, `DedicatedServer/Mirroring/ActiveOrderTracker.cs`, `SimsOnline.Client/MirrorCoordinator.cs`, `SimsOnline.Client/Program.cs`, and `SimsOnlineServerHostMod/ServerHostMain.cs`.

## Related pages

[Autonomy](autonomy.md), [refusal reasons](refusal-reasons.md), [routing contract](routing.md), [simulation overview](../simulation.md), and [protocol](../protocol.md).
