# Recording and replay boundaries

The presentation system records selected operations at shared engine boundaries and replays those operations on a client-role game. It does not record an entire gameplay interaction as a deterministic movie. A server interaction still runs on the authoritative server; the client receives selected visible operations, object/identity updates, snapshots, and interaction-replay instructions according to separate paths.

## Two meanings of replay

“Replay” is overloaded in this repository. Keep the two paths distinct:

| Path | Input | Client action | Purpose |
|---|---|---|---|
| SACS presentation replay | Ordered animation/SACS boundary lines | Apply SACS calls to the local game | Reproduce selected animation state/visual operations |
| Interaction replay | `InteractionReplayQueued` bundle with key, target and parameters | Requeue or translate an interaction according to policy | Approximate selected gameplay interactions on a client |
| Snapshot mirror | `FullSnapshot` / `SnapshotDelta` | Apply selected state such as positions | Converge observation/state fields |
| Bridge frame recording | Raw host-bridge frames | Preserve diagnostics for troubleshooting | Inspect transport and handler behavior; not a replay contract |

A client can have SACS presentation replay enabled while interaction replay is disabled. `SIMSONLINE_MIRROR_INTERACTIONS=0` turns off interaction replay without disabling positions, aliases, spawns and state; the setting is read in `SimsOnline.Client/Program.cs` and is a diagnostic/configuration boundary, not a public protocol mode.

## Boundary principle

The host recorder wraps `ISacsProxy`, the shared animation call interface, rather than adding a logging call to each gameplay interaction. `SacsRecorder` forwards the original operation and independently records selected fields. This makes the recorder observational: if recording fails, the host operation should still be forwarded. It also avoids silently missing a producer that shares the same SACS boundary.

Current recorder modules include animation, route, object, slot, audio, VFX and look-at recorders, plus state and presentation helpers. Each module has its own record shape and replay handler. A feature name alone is not evidence that every operation of that class is captured; verify a producer and matching consumer.

## Ring and record lifecycle

The recorder ring assigns sequence values to live lines. A new mirror can begin from a current cursor instead of receiving all prior transient effects. When a driver becomes relevant after the ring's earlier acquire records, the recorder maintains bounded driver history so the client can receive the setup and current state needed for that driver.

The controller `PresentationStream` polls recorder pages, selects target drivers, and forwards batches as `ServerEvent.SimAnimation`. It limits dump pages and lines per event. The client decodes the text and passes it to `SACS_REPLAY`, which is client-role-only. Replay order matters: a driver must exist before its state changes, and later operations may depend on object/slot setup.

```text
Recorder boundary -> ordered ring -> target/interest filter -> controller pages
     -> SimAnimation events -> UTF-8/hex bridge batch -> SACS_REPLAY -> local proxy
```

This path is not a permanent event store. Ring truncation, a fresh cursor, interest scope and per-driver history all constrain which past operations a joining player receives.

## Frame limits and batching

The controller uses `MaxDumpPages`, `MaxLinesPerEvent`, and `ReplayHexBudget`. `PresentationStream.ReplayBatches` includes newline bytes when calculating the encoded payload budget. Hex encoding expands each UTF-8 byte to two characters, so batching must consider encoded size rather than line count alone.

If a line is individually too large or malformed, the decoder/replay path can reject or skip that record. Do not remove the batch bound simply to make a large operation pass; split a semantically valid operation or use a structured payload with explicit size limits.

## Replay identity and mapping

SACS lines can contain actor, target, driver or object identifiers. A `ObjectId` is local to a game process. The receiving client must resolve the server ID through aliases or replay-object mapping before applying a call. A missing prop or actor can therefore be skipped even when the frame decoded correctly.

The client may maintain a local driver handle that differs from the server's driver handle. Driver acquisition and actor mapping are setup requirements. Replaying only the most recent state line without the setup context can produce a no-op, incorrect actor or stale animation.

The snapshot path uses structured DTOs and identity fields (including optional `SimDescriptionId`); the SACS ring uses line-specific runtime IDs. Never treat a SACS `ObjectId` as a stable cross-save key.

## Ordering and transient effects

State changes can be replayed as current driver state; transient effects such as sounds, VFX or one-shot events should not be replayed merely because an old ring record remains. Current-target history and sequence ordering reduce that risk, but each recorder module still owns its replay semantics.

`MirrorCoordinator` handles public wire events in receive order. It may coalesce multiple snapshot deltas; it does not collapse arbitrary animation lines into snapshot state. Interaction `Queued` and `InteractionReplayQueued` records have a separate same-batch coalescing/batch dispatch path. Do not apply snapshot coalescing assumptions to animation/SACS records.

## Failure taxonomy

| Failure stage | Example | Diagnostic interpretation |
|---|---|---|
| Producer | No line captured | Operation bypassed recorder, target not captured, ring disabled, or unsupported module |
| Ring/poll | Producer line exists but no event | Cursor, page limit, target scope, or polling issue |
| Controller event | Event not delivered to client | Connection/interest/queue delivery issue |
| Decode | Invalid line or hex | Wire corruption, incompatible format, or malformed producer record |
| Identity | Actor/driver/object not found | Alias or local-world object mapping incomplete |
| Replay | Bridge handler skips/rejects call | Role gate, missing proxy, invalid arity, or unsupported operation |
| Visual result | Call accepted, image differs | Animation resources, posture context, world state, or camera evidence needed |
| Interaction result | Client Sim doesn't reproduce server logic | Expected: presentation replay is not deterministic gameplay simulation |

“Replay succeeded” must name the layer: decoded, delivered, handler accepted, call applied, or visually matched. Each is a different claim.

## Snapshot versus recorded history

A `SimulationSnapshot` is current selected state. It is not a history and cannot reconstruct every intermediate transition. Conversely, a SACS ring history is not a full state snapshot: it may omit current world properties, Sim descriptions, ownership, lot topology, or object contents. A reliable join combines the appropriate snapshot baseline with relevant active-driver presentation history, rather than substituting one for the other.

`ForceFull` means the server is asking the client to reset from a full snapshot. It does not mean “replay every historical SACS event.” The client drains stale delta backlog before applying the authoritative full snapshot; active presentation history remains governed by its own target/cursor path.

## Interaction replay classification

`InteractionReplayPolicyResolver` classifies captured interaction bundles into policies such as:

- `NeverReplay` for reactions or unsafe/unowned behavior;
- `ReplayAsRoute` for a routing interaction with a destination;
- `ReplayExact` for a supported terminal interaction with a target;
- `ForceAlign` for selected posture transitions with a known container;
- slide-only or no replay for an unknown action without a safe target.

The policy resolver is not the SACS recorder. For example, an interaction might fail to execute on a client while its animation SACS calls still render, or an interaction replay may run while SACS recording is disabled.

### Trace missing interaction observations

| Boundary | What controls the observation |
|---|---|
| Client input capture | `ClientCaptureFilter` checks which local interactions can become capture requests. See [Capture and input](capture-input-pipeline.md). |
| Host autonomy polling | `UnclaimedRoutingEmitFilter` limits observations to Sims on lots in active interest scope that are neither claimed nor covered by the legacy autonomy watch. The interaction must be routing or target a world object. `PollUnclaimedRoutingEmit` also skips a current interaction with `UserDirected` priority. |
| Claimed-Sim seating | `SitTransitionEmitFilter` recognizes a sit in the transition slot. The host can emit it when the current interaction slot holds a different action, and tracks the most recently emitted sit ID per actor to avoid repeats. |
| Controller translation | `InteractionReplayPolicyResolver` selects a policy using the recorded interaction, destination or target. Check this stage when the host emitted a queue observation and the client received no replay action. |

Use the fully qualified interaction key when tracing route or terminal classification. Different interaction types can share the same short name.

## Verification and tests

`DedicatedServer.Tests/Program.cs` covers `MirrorEngine.GoldenReplayFromRecordedRun3`, replay policy classifications, parser fixtures, `MirrorEngine.InteractionReplaySwitchDropsOnlyInteractionReplay`, replay batch dispatch, and recorder-related line parsing. `DedicatedServer.Tests/JoinTraceTests.cs` validates live/history ordering for presentation stream events. `DedicatedServer.Tests/CoordinatorTimerTests.cs` covers receive/drain ordering and coalescing paths.

Useful code entry points:

- `SimsOnlineServerHostMod/SacsRecorder.cs` — `ISacsProxy` decorator, ring and driver history.
- `SimsOnlineServerHostMod/AnimationRecorder.cs`, `RouteRecorder.cs`, `ObjectsRecorder.cs`, `SlotsRecorder.cs`, `AudioRecorder.cs`, `VfxRecorder.cs`, `LookAtRecorder.cs` — producer modules.
- `DedicatedServer/Mirroring/PresentationStream.cs` — targets, limits, paging and encoding.
- `DedicatedServer/Authority/AuthoritativeSession.cs` — event/history orchestration.
- `SimsOnlineServerHostMod/ServerHostMain.cs` — `SACS_RECORDER` and `SACS_REPLAY` bridge handlers.
- `DedicatedServer/Mirroring/InteractionReplayPolicyResolver.cs` — interaction-replay classification.

Automated tests prove format and controller behavior. A live visual comparison is required to claim that a particular animation or effect appeared correctly on a particular build and save. See [Presentation scenarios](presentation-scenarios.md) for a bounded scenario matrix.

## Replay handler contract

`HandleSacsReplay` is a client-role-only bridge handler. It decodes replay text into lines and calls `ApplyReplayBatch`; it rejects missing proxy/payload or an invalid role. The return status should be read alongside per-line skip counters: a batch command can be accepted while individual records are skipped because a target is missing or a line is unsupported.

Bridge frame capture (`--bridge-record`) is a separate diagnostic facility. In client host mode it appends timestamped bridge frames through `FrameRecord`; this captures what crossed the bridge but is not a supported replay archive format. The file may contain user-originated text or object IDs, so keep it local and redact it before sharing.

## Buffer and target policy

The recorder ring is bounded and the controller reads pages rather than draining an unlimited history in one tick. Target selection begins with claimed Sims and adds Sims within the active interest lots. A late join does not receive animation history for every Sim in the world. The target set is part of privacy and bandwidth scope as well as a performance control.

When a client changes interest scope, it may need a fresh active-driver history for newly relevant Sims. A target that was omitted earlier is not proof that its recorder never captured a line. Conversely, broadening target scope without explicit limits risks transmitting unrelated world activity.

## What replay can and cannot tell you

A SACS replay line can establish that a specific call with specific arguments was observed and attempted on the client. A `SACS_REPLAY` success can establish that the handler processed the batch. Neither proves:

- the local game accepted every call's semantic effect;
- the corresponding mesh/clip visually matched the server;
- the target object had the same state or placement;
- the interaction logic ran on the client;
- the client world will persist that result after load;
- every engine operation in that gameplay action was captured.

For visual evaluation, sample several fields separately: animation state, position at rest, held object, posture, current interaction and visible props. An animation machine state match is not interchangeable with interaction-object match.

## Safe extension checklist

When adding a recorder or replay line:

1. Name the shared producer boundary and prove the operation passes through it.
2. Define stable line arity, sequence handling and value encoding.
3. Decide whether the line is persistent current state or transient effect.
4. Identify every process-local object/driver id and map it on the client.
5. Bound line length, page size and aggregate bridge payload.
6. Add malformed-line and unsupported-operation behavior that skips safely.
7. Preserve per-driver ordering and avoid replay across a stale setup boundary.
8. Add a parser/encoder test and host/client role-gate test.
9. Add a live visual scenario for the targeted operation.
10. Check that ring history does not retain personal/free-form data longer than needed.

## Related diagnostics

If SACS counters report applied records but visuals differ, check the local animation resource/build and whether a second local system writes state after replay. If positions drift, inspect snapshot filtering and active orders. If a target is skipped, inspect aliases and interest scope before changing the animation serialization. For exact reproducible comparisons, see [Presentation scenarios](presentation-scenarios.md).

## Interaction state is not presentation history

The public interaction event pipeline may classify or replay supported queue entries, while the SACS recorder captures the actual animation boundary. One can succeed without the other. For instance, an interaction may queue on the client but produce a different animation because the local object or game build differs; an animation may replay onto a puppet without a corresponding local `Interaction` object.

When adding instrumentation, label the source: `interaction_replay_queued` means a gameplay requeue request, while `sim_animation` means recorded presentation lines. Neither should be renamed into a generic “replay event” in documentation or logs.

## Data retention and shared traces

Ring history and bridge frame capture can contain player-created labels or object names. Ring buffers should remain bounded; frame captures should only be enabled for a specific diagnostic window. Public examples should use synthetic names, ids and text. Keep raw world/save payloads out of issue reports.
