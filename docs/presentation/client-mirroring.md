# Client mirroring and resynchronization

Client mirroring applies selected server events to another game process so that the player can observe shared activity. The receive path, state path, interaction path and animation path have distinct ordering and recovery rules. The controller does not treat the client's world as authoritative merely because it can apply an event there.

## Receive and dispatch pipeline

`SimsOnline.Client.ReceiveQueue` decouples TCP reads from potentially slow local bridge dispatch. The socket reader writes decoded event lines to an in-order channel; `MirrorCoordinator` drains and dispatches them. This protects the network reader from a slow game command while preserving event order for the mirror.

```text
TCP line reader -> in-order ReceiveQueue -> MirrorCoordinator
  -> decode/filter -> coalesce selected snapshot deltas -> MirrorEngine
  -> EventToBridgeCommandTranslator -> client HostBridge -> local game mod
```

Other work (such as timer-triggered operations) is posted onto the drain loop so it runs in channel order rather than racing a bridge apply from a different thread.

## What the public snapshot carries

`SimulationSnapshot` carries selected Sim and object records, tick, world/save names and speed/time metadata. A `SimSnapshot` includes fields used by the current mirror such as object ID, position, lot ID, current interaction key, and optional identity/household data. The record is intentionally not an entire Sims 3 object graph.

Snapshot application updates supported state such as positions and object state through the mirror translator. Gameplay-owned values not represented by the public snapshot are not implicitly synchronized. See [Snapshot fields](../data-and-persistence/snapshot-fields.md) for field meanings and identity caveats.

## Per-client self filtering

`MirrorEventFilter.ShouldSkip` skips an `InteractionQueued` event when its actor is owned by this client: the local player already issued that command, so replaying the authoritative echo could duplicate the local order. Remote-owned actors remain eligible for mirror application.

Snapshots can mix owned and remote Sims, so the client filters individual `SimSnapshot` rows rather than dropping the whole event. `ComputePositionMirrorStripSet` intersects the client's owned Sim set with active directed orders. An owned Sim actively routing is temporarily omitted from position writes so its local route can run; idle owned Sims and remote-owned Sims remain eligible according to the active mirror mode.

Autonomy changes are not blanket-skipped for self-owned Sims: the host remains the sole authority for the mirrored session, and local AI may need suppression even on a Sim the player directs. The authority mode and interaction replay configuration affect which owned rows are mirrored.

## Delta coalescing

`MirrorCoordinator.Merge` combines a batch of `SimulationSnapshot` records at whole-record granularity. For each Sim or object id, the newest record wins; fields are not merged individually. The resulting tick, speed, world name and save name come from the highest-tick snapshot in the batch.

Events other than snapshot deltas are not merged across types. The coordinator preserves their receive order. Multiple interaction queued/replay records can use dedicated batch coalescing, with chain-specific logic; that is not general event compaction.

Why whole-record replacement matters: if a newer record omits an optional field, splicing an older field into it could synthesize a state no server tick ever reported. The design prefers a coherent newest record over a field-wise “best of both” object.

The receive channel is unbounded; coalescing happens on the reader/drain side. It does not reduce bytes that have already crossed TCP, and it is not permission to drop a non-snapshot event.

## Tick gaps and drift

The client tracks raw snapshot-delta tick contiguity for drift detection before coalescing. A gap can indicate that the local mirror has missed an update sequence; the highest coalesced endpoint alone cannot reveal whether there was a gap. `MirrorCoordinator` owns the single debounced `RequestSnapshot` path, avoiding duplicate full-snapshot requests from multiple handlers.

A `FullSnapshot` is an authoritative reset baseline. `SnapshotDelta.ForceFull` marks a server-directed full-resync event; when received, the coordinator drains/discards stale accumulated delta backlog and applies the authoritative reset. While a server full snapshot is in flight, client-side drift requests are suppressed so both sides do not request overlapping resyncs.

`ResyncPolicy` decides when the server sends `ForceFull` based on a valid acknowledgement baseline and per-player lag. It targets only the lagging player. A force-full request is not a save operation and does not restore durable game data.

## Snapshot acknowledgements

The mirror dispatch path acknowledges the highest applied tick after successful dispatch. That acknowledgement helps the server estimate client lag. If bridge application fails, telemetry and log classification should distinguish partial dispatch from complete application; receiving a line is not the same as applying it.

A valid ack baseline matters. The server avoids forcing a full snapshot when it lacks a meaningful acknowledgement baseline, and tests pin that behavior. A stale ack should not be interpreted as a complete picture of which specific Sim rows were applied.

## Event stream versus state stream

| Stream | Examples | Coalesced? | Recovery |
|---|---|---:|---|
| Snapshot state | Full snapshot, sparse delta | Yes, selected consecutive deltas | Request/receive full snapshot |
| Interaction intent/replay | Queue order, terminal interaction | Only dedicated same-batch policy | Retry/resync depends on operation semantics |
| Presentation history | SACS animation lines | Ordered, bounded batches | Targeted driver history or fresh cursor |
| World clock | `WorldTimeChanged` | No generic folding | Latest authoritative event/snapshot time |
| Dialog answer | `AnswerDialog` / captured event | No | Dialog timeout/default |
| Menu refusal | `Rf` presentation line | No generic state merge | Explicit clear or TTL |

State can be corrected from a newer snapshot. A one-shot event cannot always be reconstructed from current state, so the coordinator preserves ordering rather than folding all event types into a state object.

## Applying state is not executing simulation

A position write can correct drift but does not plan a route. An interaction replay can approximate a supported command but may not have the same object graph, inventory contents or autonomy result. A presentation replay can set an animation state without creating the corresponding gameplay `Interaction` object. The client therefore displays a replicated/puppet view, not a second authority with proven deterministic equivalence.

If a client shows the right position but the wrong posture, inspect the route/interaction/presentation stream separately. If the Sim appears frozen, distinguish suppressed autonomy from a stuck bridge dispatch, a self-order echo, an active-order position strip, and an unavailable object target.

## Failure and telemetry boundaries

`SimsOnline.Client.Program` classifies mirror dispatch failures, including target resolution failures and partial bridge dispatch. A per-command failure can occur after other commands from the same event succeeded. Avoid saying “the snapshot failed” when only one target-resolution operation failed.

Recommended diagnostic sequence:

1. Confirm the event arrived at the client receive queue.
2. Confirm decode and protocol type.
3. Check self/interest filtering decision.
4. Check coalescing and tick gap decision.
5. Check translated bridge commands and order.
6. Check role gate and client bridge response.
7. Check local object/Sim resolution.
8. Check applied snapshot and final local state.

Logs should use the existing typed failure categories and avoid dumping household/save data or private object content.

## Tests and source map

`DedicatedServer.Tests/CoordinatorTimerTests.cs` covers `MirrorCoordinator.TimerFlushWaitsForTheDrainLoop`, `PostedWorkRunsInLineOrder`, and join drain ordering. `DedicatedServer.Tests/Program.cs` tests coalescing, `ForceFull`, filters, active-order stripping, replay policy, and target translation. `DedicatedServer.Tests/WireJsonPinningTests.cs` pins `SnapshotDelta.ForceFull` serialization.

Key implementation files:

- `SimsOnline.Client/ReceiveQueue.cs` — network-reader queue.
- `SimsOnline.Client/MirrorCoordinator.cs` — drain, coalescing, ordering, resync requests and ForceFull.
- `DedicatedServer/Mirroring/MirrorEventFilter.cs` — ownership filtering.
- `DedicatedServer/Mirroring/MirrorEngine.cs` — translate and apply event commands.
- `DedicatedServer/Mirroring/EventToBridgeCommandTranslator.cs` — supported state/action translation.
- `DedicatedServer/Authority/ResyncPolicy.cs` and `DedicatedServer/Network/ClientListenerServer.cs` — server lag policy and per-player full-snapshot delivery.

For animation line capture, see [Recording and replay](recording-and-replay.md). For the local game process and role separation, see [Client game runtime](client-vm.md).

## Ordering rules in the drain loop

The coordinator greedily drains up to `MaxFoldPerCycle`; it may briefly read ahead when the first event is an interaction so same-window commands can be batched. During batch processing:

- snapshot deltas fold by speed group and whole-record latest tick;
- world-time events can retain only the latest pending time while snapshots apply;
- interaction queue events pass through the chain-aware coalescer;
- replay-interaction events remain ordered for their own batch translator;
- consecutive `SimAnimation` events can be joined into one event dispatch while each raw input still advances `afterLine` accounting;
- other events flush pending coalesced work before they are dispatched.

The ordering decisions are semantic. A dialog answer, claim change, or interaction terminal event should not be folded into an unrelated snapshot just to reduce bridge round trips.

## `ForceFull` and the snapshot backlog

The server's `ForceFull` event is a discontinuity marker: queued sparse deltas before it no longer form the right baseline. The client discards that stale backlog, clears the in-flight resync suppression state, and dispatches the full snapshot directly. A normal delta gap is not by itself a reason to clear all event types; the full resync path is explicit.

A dangerous regression would apply old deltas after the full snapshot and move a Sim backwards. Another would keep requesting snapshots while the server is already sending its full response. Tests should assert both backlog discard and single-request arbitration.

## Performance without event loss

`MaxFoldPerCycle` bounds work in one drain cycle; it does not imply the channel drops excess lines. `ReceiveQueue` remains unbounded by design so snapshot coalescing is reader-side and event records are not silently discarded at the socket boundary. This trades memory growth risk under prolonged bridge stalls for no protocol-level data loss; operational backpressure and failure reporting remain important.

When a client cannot keep up, look at bridge command latency, animations queued/in-flight and snapshot delta cadence separately. If `SimAnimation` events accumulate while the bridge is busy, merge consecutive animation lines within the batch and preserve their order. Do not fold a dialog or state event into the animation batch.

## Coordinator tests worth preserving

- `MirrorCoordinator.CoalescesTenDeltasToOneDispatch` proves latest-state folding.
- `MirrorCoordinator.ForceFullDrainsBacklogAndAppliesFullSnapshot` proves authoritative reset precedence.
- `MirrorCoordinator.BatchesInteractionReplayQueuedWithinReadAhead` proves a separate interaction batch path.
- `CoordinatorTimerTests.PostedWorkRunsInLineOrder` proves timers share the dispatch loop order.
- `CoordinatorTimerTests.JoinDrainRunsPostedWorkBetweenLines` protects join-time reads.

These tests should assert observed dispatch order, not only call counts. A reduced number of bridge calls is not a correct optimization if the final state changes.

## Coalescing design table

| Incoming sequence | Dispatch behavior |
|---|---|
| Delta, delta, delta | Fold per speed group and dispatch latest whole records |
| Delta, interaction, delta | Flush pending state before the interaction; preserve event boundary |
| Several same-batch queue intents | Chain-aware interaction coalescer, then ordered dispatch |
| Several replay interaction events | Batch using replay-specific translator |
| Consecutive animation events | Join line lists in order to reduce bridge round trips |
| ForceFull snapshot | Discard obsolete delta backlog, apply reset directly |
| Posted timer work marker | Execute on the drain loop between wire lines |

This explains why the coordinator is more than a generic queue: it reduces redundant state writes while keeping non-state operations ordered and correlated.

## `DispatchResult` and partial application

`MirrorEngine.DispatchAsync` translates an event to one or more bridge commands and collects command failures. One event can contain several position/object updates; a successful command for one target does not imply every target was resolved. The client logs partial failures and distinguishes target resolution from general dispatch failure.

| Observation | Interpretation |
|---|---|
| Event decoded, zero bridge commands | Event type is status-only, filtered, or has no supported translation |
| Bridge command accepted | Handler accepted/scheduled the command; readback may still be pending |
| One command fails in a batch | Partial application; other commands may have succeeded |
| Target resolution failure | Client alias/world counterpart missing |
| RequestSnapshot sent | Client asked for state baseline; no guarantee response arrived yet |
| Ack tick advanced | Dispatch path completed through its acknowledgement hook |

Do not return early from an event on its first missing target if independent updates can still be applied safely. At the same time, do not acknowledge a partially applied state as complete unless the policy explicitly defines that behavior.

## Interaction replay and snapshot are separate controls

`InteractionReplayEnabled` controls the interaction replay translator. Snapshot position deltas still flow when interaction replay is off. The presentation-only configuration can therefore have the server's interaction and position on the client while the local interaction object does not exist. This is a deliberate separation between the appearance state and the gameplay queue.

The event filter also has stateful context from owned Sim IDs and active directed orders. When an order completes or fails, `OrderCompletionReassert` can reapply the latest server position that was intentionally withheld during the order. The reassertion avoids leaving a Sim permanently excluded after its local route ends.

## Snapshot consistency considerations

A snapshot delta is sparse. If the client folds several deltas, the merged result must preserve the latest full `SimSnapshot` record for each runtime ID. If an event boundary occurs between two deltas, pending state is flushed before the event; this keeps the non-state event ordered relative to the state it preceded/followed.

The controller's tick-contiguity check uses raw deltas before merge. A merged snapshot at tick 120 may conceal skipped raw tick 117; therefore drift logic cannot inspect only the merged endpoint. `MirrorCoordinator` intentionally separates coalescing from contiguity tracking.
