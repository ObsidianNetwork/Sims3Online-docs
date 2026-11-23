# Simulation snapshot fields

`SimulationSnapshot` is the current typed protocol representation of selected simulation state. It is not a native save, and the record definition does not promise a transactional view of every game subsystem. Use its tick and identity context to interpret observations, and treat absent optional values as unknown rather than as a command to clear local state.

## Record shape

The C# records are defined in `SimsOnline.Protocol/SimulationSnapshot.cs`:

```csharp
public sealed record SimulationSnapshot(
    long Tick,
    string WorldName,
    string SaveName,
    GameSpeed Speed,
    IReadOnlyList<SimSnapshot> Sims,
    IReadOnlyList<ObjectSnapshot> Objects);
```

Each `SimSnapshot` has required `ObjectId`, `Name`, `HouseholdId`, and `Position`, plus nullable interaction and mirror metadata. `ObjectSnapshot` contains `ObjectId`, `ScriptClass`, `Position`, and nullable `LotId`.

| Sim field | C# type | Interpretation |
|---|---|---|
| `ObjectId` | `ulong` | Runtime object identity in the producing loaded game |
| `Name` | `string` | Display name; not a unique identifier |
| `HouseholdId` | `ulong` | Household identifier available from the producer |
| `Position` | `Vector3Dto` | Point-in-time `X`, `Y`, `Z` position |
| `CurrentInteractionId` | `ulong?` | Optional runtime interaction identity |
| `OwningPlayerId` | `int?` | Optional owner assignment; null is unknown/not supplied |
| `CurrentInteractionKey` | `string?` | Optional descriptive interaction key |
| `LotId` | `ulong?` | Optional lot scope |
| `InteractionQueueCount` | `int?` | Optional count only; it is not the queue content |
| `PostureContainerObjectId` | `ulong?` | Optional object/container identity |
| `PostureType` | `string?` | Optional posture label, not a transition history |
| `SimDescriptionId` | `ulong?` | Optional cross-process identity for save-born descriptions |

`ObjectSnapshot` has no inventory or current-interaction field. `SimSnapshot` has no full queue, motive set, relationship graph, route history, or animation state machine. Those omissions are deliberate boundaries of this record type, not zero-valued state.

## Identity rules

`ObjectId` refers to a runtime object in the loaded save. Two separate game processes can instantiate equivalent objects with different runtime object IDs. `SimDescriptionId` is the cross-process identity field when present; `SimDescriptionId` is nullable so consumers must tolerate older or incomplete producers. Resolve the logical Sim locally before operating on its local game object. Never use a display name as a key.

A posture container ID and the Sim's own object ID are from the producer's object namespace. The consumer must map them to local client objects before using them. Equal posture labels do not imply equal animation state, seat slot, or interaction lifecycle.

## Snapshot versus delta

The public event model distinguishes `ServerEvent.FullSnapshot` and `ServerEvent.SnapshotDelta`. A snapshot is a baseline-like selected state view; a delta is not a complete object graph. `SnapshotDelta` also carries optional `ForceFull`, defaulting to `false`; consumers that implement this field use `true` to replace the prior mirror rather than merge as an ordinary delta. The event declarations are in `SimsOnline.Protocol/ServerEvent.cs`.

A consumer should apply only data that belongs to the expected session/world and is newer than its current baseline. A missing optional field means â€œnot supplied,â€ not â€œclear the local value.â€ A snapshot can repair current selected fields, but cannot recreate a one-shot action that occurred before the snapshot.

## Reading example

This is a schematic view of the record contract, not a complete wire payload:

```text
SimulationSnapshot
  Tick: 8123
  WorldName: "Sunset Valley"
  SaveName: "Town"
  Speed: Normal
  Sims[0]: ObjectId=9001, SimDescriptionId=55012, LotId=71,
           CurrentInteractionKey=null, InteractionQueueCount=2,
           PostureType="Seated", PostureContainerObjectId=700
```

A different client might report the same `SimDescriptionId` but a different runtime `ObjectId` and local `PostureContainerObjectId`. That difference is not by itself a desynchronization. Compare mapped identity and compatible fields at nearby ticks.

## Consumer validation

- Check that the snapshot belongs to the expected session/world and reject stale tick data.
- Validate that numeric position values are finite before using them to place or animate a local object.
- Bound processing at the framing/transport layer; this record type does not itself impose an object-count limit.
- Treat `null` for owner, interaction, posture, lot, and stable identity as unknown/absent.
- Pair `CurrentInteractionId` with its interaction key and context; an ID alone is not replayable.
- Use `InteractionQueueCount` only as a count. It cannot describe ordering or contents.
- Do not treat a state correction as a gameplay command.

## Failure modes

| Mistake | Consequence |
|---|---|
| Match by `Name` | Colliding names can select the wrong Sim |
| Reuse host `ObjectId` in the client process | A valid logical Sim may not resolve locally |
| Treat null posture as â€œstandingâ€ | Unsupported producer data can clear a valid local pose |
| Treat queue count as queue entries | No action identity or ordering can be recovered |
| Infer inventory from an object snapshot | `ObjectSnapshot` does not carry inventory membership |
| Treat snapshot equality as visual equality | Animation, routing, slot state, and effects are outside these fields |

## Verification and source map

`SimsOnline.Protocol/SimulationSnapshot.cs` is the schema source. `DedicatedServer.Tests/WireJsonPinningTests.cs` pins selected public protocol shapes; `DedicatedServer.Tests/JoinTraceTests.cs` exercises join-time delivery ordering. A codec test proves serialization only. For a live mirror claim, compare corresponding host and client instances after resolving identities and sampling near the same tick.

Related: [Persistence overview](overview.md), [Protocol](../protocol.md), [Bridge and formats](../bridge-and-formats.md), [Testing](../testing.md).

## Field-level wire compatibility

`SimulationSnapshot` uses `System.Text.Json` through the public polymorphic protocol. `SimulationSnapshot.cs` defines the record shape; `ServerEvent.FullSnapshot` and `ServerEvent.SnapshotDelta` wrap it. The wire uses camel-case property names under the configured serializer, for example `simDescriptionId`, `interactionQueueCount`, and `postureContainerObjectId`.

A schematic event with the current field names is:

```json
{
  "$type": "full_snapshot",
  "snapshot": {
    "tick": 8123,
    "worldName": "Sunset Valley",
    "saveName": "Neighborhood",
    "speed": "Normal",
    "sims": [{
      "objectId": 9001,
      "name": "Example Sim",
      "householdId": 84,
      "position": { "x": 1.25, "y": 2.0, "z": 3.0 },
      "currentInteractionId": null,
      "owningPlayerId": 4,
      "currentInteractionKey": "Terrain.GoHere",
      "lotId": 71,
      "interactionQueueCount": 2,
      "postureContainerObjectId": null,
      "postureType": null,
      "simDescriptionId": 55012
    }],
    "objects": []
  }
}
```

This is a hand-readable representation of the current DTO contract; use `WireJsonPinningTests.cs` for canonical escaping, discriminator, enum and omission behavior. The JSON sample's numeric identities are fabricated and do not identify a real save.

`CurrentInteractionId`, `OwningPlayerId`, `CurrentInteractionKey`, `LotId`, `InteractionQueueCount`, posture fields, and `SimDescriptionId` are nullable. `ObjectId`, `Name`, `HouseholdId`, and `Position` are required positional-record arguments in the C# model. A compatibility reader should not assume that old producers emit every nullable field.

## Identity mapping example

An identity comparison must not compare runtime IDs directly across two independent games:

| Host field | Client field | Interpretation |
|---|---|---|
| `SimDescriptionId = 55012` | `SimDescriptionId = 55012` | Likely same save-born Sim identity |
| `ObjectId = 9001` | `ObjectId = 1310` | Different process-local runtime IDs are expected |
| `PostureContainerObjectId = 700` | `PostureContainerObjectId = 998` | Map each container locally before judging the pose |
| `Name = "Example Sim"` | `Name = "Example Sim"` | Display agreement, not identity proof |

`SimDescriptionId` is optional and its XML documentation explicitly allows null from an older mod. If it is absent, a caller needs another established alias mapping; matching by name is not a safe fallback.

## Full snapshot, delta, and force-full semantics

`SnapshotDelta(SimulationSnapshot Snapshot, bool ForceFull = false)` carries `ForceFull` additively. The default is false for older payloads. In `MirrorCoordinator`, a `ForceFull` delta drains the available input backlog, discards coalesced position state, clears the in-flight resync marker, and dispatches that event as a full replacement. That behavior prevents a stale backlog from being applied after the server has declared that incremental state is no longer a valid baseline.

| Event | Consumer model |
|---|---|
| `FullSnapshot` | Establish/re-establish the selected current-state baseline |
| `SnapshotDelta` with `ForceFull == false` | Coalesce/apply changes relative to existing state |
| `SnapshotDelta` with `ForceFull == true` | Drain stale buffered work and reset from this snapshot |
| `RequestSnapshot` | Ask server for a fresh observation; not itself a state change |
| `AckSnapshot` | Confirm a client-applied snapshot tick to the session |

A snapshot tick is a simulation ordering value, not UTC time. `AckSnapshot` is per-player and carries the tick, as documented on `ClientCommand.AckSnapshot`.

## Validation and application policy

`SimulationSnapshot` is a data record, not a validator. Consumers should separately validate numeric content and application preconditions. For example, `float.NaN` and infinities are legal bit-level float values but nonsensical positions; guard them before spatial writes. Avoid indexing an array by `ObjectId`, and avoid using external strings as file paths.

| Field group | Safe consumer treatment |
|---|---|
| Tick/session identity | Reject stale updates and updates for a prior world/session |
| World/save names | Diagnostic display only; never trust as a path |
| Speed | Validate known `GameSpeed` members before forwarding to game |
| Position | Require finite coordinates and a resolved local Sim |
| Lot IDs | Use for interest/scope filtering only after validating identity |
| Interaction fields | Display/diagnostic data; not a replay command by themselves |
| Posture fields | Optional hints; resolve local container and supporting presentation |
| Queue count | Non-negative coarse metric, never substitute for queue contents |
| Stable Sim identity | Prefer when present but maintain fallback alias policy |

Do not interpret missing data as a delete. Removal requires an explicit lifecycle event or a full replacement whose semantics define omission as absence.

## Snapshot consistency is bounded by the producer

`SimulationSnapshot` bundles a tick and lists, but the record type alone does not promise every engine field was sampled in one atomic operation. If two rows are read at slightly different points during a tick, the consumer still sees one envelope. For position/motion comparison, pair close samples and account for active routes. For durable-state comparison, save and reload instead of relying on the observation snapshot.

`ServerEvent.SnapshotDelta.ForceFull` also does not change the type of its inner `SimulationSnapshot`. It changes how the receiving mirror treats the delta. A sender adding new fields should retain additive compatibility and pin them in `WireJsonPinningTests`.

## Related tests and observed failure modes

- `WireJsonPinningTests.cs` has exact JSON pins for `snapshot_delta`, `full_snapshot`, `set_world_time`, and `submit_lot_export`.
- `JoinTraceTests.cs` validates the initial join snapshot and its delivery order relative to `SessionJoined`.
- `DedicatedServer.Tests/ClientReceiveDrainTests.cs` and `SimsOnline.Client/MirrorCoordinator.cs` cover receive draining and mirror coordination.
- `SimsOnlineServerHostMod/SelectionGuard.cs` plus `SelectionGuardTests.cs` protect active-player selection from replay behavior.

The classic false-positive is two objects with equal names and different identity; another is a posture label that matches while the corresponding local animation driver does not. Both require process-local mapping and evidence from the appropriate mirror subsystem.

## Field lifecycle: identity and ownership

A snapshot `ObjectId` is the runtime identity of an object observed by the current host. In the same process, it is useful for lookups and incremental updates. Across distinct game instances, it may differ. `SimDescriptionId` is optional and is the more useful cross-instance Sim identity when present; it identifies a Sim description, not every runtime object or every save-generation mapping.

| Identity | Scope | Appropriate use |
|---|---|---|
| `ObjectId` | One loaded game/process | Runtime event and object lookup |
| `SimDescriptionId` | Sim-description identity when available | Alias/cross-process matching for a Sim |
| `HouseholdId` | One world/save identity context | Household-level details when the mapping is known |
| `LotId` | Loaded world/lot identity | Interest scope and export validation |
| `bookId` | Book definition/resource identity | Resolve the corresponding book definition, not an instance |

Do not use a zero or missing optional identifier as a stable key. The server may not know a cross-process alias yet, especially during initial world bootstrap; spawn/alias events and full snapshots establish those mappings over time.

Ownership is not a field that can be inferred from `ObjectId`. A Sim belongs to a player because the authoritative session claim table says so. A snapshot can include Sims outside the local player's claims because they are remote puppets.

## Snapshot versus delta semantics

`FullSnapshot` gives the receiver a baseline. `SnapshotDelta` contains changed records at a later tick; omitted Sims are not automatically deleted, and omitted optional values do not mean the field should be cleared unless the DTO explicitly defines that behavior. Entity removals are handled by their lifecycle events, not by treating every sparse list as a complete roster.

`SnapshotDelta.ForceFull` is a resynchronization marker for client application. On that path `MirrorCoordinator` discards stale backlog and applies the full authoritative snapshot. It does not change the meaning of individual `SimSnapshot` fields and is not a durable save restore.

The `MirrorCoordinator.Merge` function uses whole-record latest-tick-wins logic for duplicate Sim/Object IDs. It does not merge a position from one record with an interaction key from an older record. As a result, consumers should preserve the complete record when adding a field; it should not be split into ad hoc per-field caches unless a separate protocol explicitly requires that.

## Presence and missing records

A missing Sim row in a sparse delta is normally â€œnot changed in this delta,â€ not â€œthe Sim was deleted.â€ A Sim can be absent because it is outside the current interest scope, was filtered from this client's position mirror, has not yet been spawned locally, or was removed from the world. Check the event type and scope before interpreting absence.

`MirrorEventFilter.FilterSnapshotSims` can intentionally remove self-owned Sim rows before the local mirror dispatch. That filtering is a client-side application decision; the original server snapshot can still include the Sim. A debug log of the filtered event should identify this transformation so it is not misdiagnosed as a server omission.

When a Sim/object truly leaves the world, the presence/despawn/lifecycle channel is the appropriate source. A full snapshot can reset state, but it does not turn a sparse delta into an implicit deletion protocol.

## Check snapshot population during world load

Launch a fresh standard Sims 3 host with the dedicated server's `--probe-sim-lifecycle` option. The controller requests its first snapshot and then sends `PROBE_SIM_LIFECYCLE`. The report prints the simulation tick and three counts for the world-load callback, loading-screen disposal, the first snapshot request, and the time of the probe.

| Count | What it measures |
|---|---|
| `actors` | Instantiated Sims from the host's actor list |
| `visible` | Sims in the host's visible-actor list |
| `snapshotSims` | Sims the host can encode, or the encoded Sim count for the first snapshot |

Compare `snapshotSims` at loading-screen disposal, the first request, and the decoded controller snapshot. The probe requires those counts to agree and be positive. A difference at the earlier world-load callback is reported separately. These counts measure snapshot population on the tested load. Use the [save-load readiness checks](save-load-pipeline.md#readiness-is-more-than-file-arrival) when verifying a join.

## Check posture on the host and in the snapshot

For a selected Sim, run `python scripts/bridgectl.py state <sim-id>` against the game bridge you want to inspect. Use the Sim's runtime object ID for that process. The live report includes `postureType` and `postureContainerObjectId`. To verify what traveled across the host bridge, inspect the same fields in a decoded [SNAPSHOT2 record](../protocol-reference/snapshots-and-resync.md#snapshot2-binary-layout) sampled at a nearby tick. If the container comes from another game instance, resolve it using the local identity mapping before comparing seats.

## `ForceFull` example

A client receives deltas at ticks 100, 101, 102, then is delayed while ticks 103â€“110 queue. If server lag policy sends `ForceFull`, `MirrorCoordinator` drains the old delta backlog, applies the authoritative full snapshot, and acknowledges the applied tick. It does not first replay 103â€“110 and then apply a potentially older full state. The old deltas are superseded by the full baseline.

This behavior is tested by `MirrorCoordinator.ForceFullDrainsBacklogAndAppliesFullSnapshot` and wire-pinned by `SnapshotDelta.ForceFull.RoundTripsViaCodec`. A valid ack baseline is necessary before server-side lag policy can reason about the client's progress.

## Versioning and evolution

Adding an optional snapshot field requires checking all consumers: protocol serialization, bridge translation, mirror application, logging, and tests. An additive optional DTO field can still change semantics if clients interpret missing as zero or default. Keep the default behavior explicit and pin JSON shape where wire compatibility matters.

The protocol does not promise that arbitrary unknown snapshot extensions will be applied by older clients. Compatibility requires a fallback behavior or version gate when the new field is required for correctness. Do not silently overload an existing field with a new meaning.

A `SimulationSnapshot` is also not a schema for native saves. A save-format change belongs to the game and DBPF lifecycle, while snapshot DTO versioning belongs to the network contract.

## Test design checklist

For each new or changed field, tests should cover:

- JSON encode/decode with the field set and unset;
- zero, empty and boundary values where meaningful;
- `FullSnapshot` and `SnapshotDelta` translation;
- coalescing of two ticks for the same `ObjectId`;
- self-owned/remote-owned filtering behavior;
- object/Sim alias missing and later-resolved cases;
- `ForceFull` interaction if it changes reset behavior;
- a host bridge test if the field is applied locally;
- a live cold-load test only if making a durability claim.

`DedicatedServer.Tests/WireJsonPinningTests.cs`, `DedicatedServer.Tests/CoordinatorTimerTests.cs`, and mirror tests in `DedicatedServer.Tests/Program.cs` cover current codec and application contracts.
