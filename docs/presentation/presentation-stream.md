# Presentation stream

The host game records selected animation operations at the SACS interface boundary; the server controller forwards ordered lines as `ServerEvent.SimAnimation`; a client-role host mod applies batches through `SACS_REPLAY`. This reproduces selected presentation behavior without rerunning the host interaction logic on the client.

## Producer boundary

`SimsOnlineServerHostMod/SacsRecorder.cs` implements `ISacsProxy` as a decorator around the game's SACS proxy. The recorder tracks driver/actor associations and recent driver history while forwarding calls to the underlying proxy. Recording failures are isolated from the game call path; recording is observational rather than a replacement for host behavior.

When ring capture is enabled, `SacsRecorder.RingLine` appends event lines. Current producer modules include `AnimationRecorder`, `RouteRecorder`, `ObjectsRecorder`, `SlotsRecorder`, `AudioRecorder`, `VfxRecorder`, `LookAtRecorder`, and state/presentation helpers under `SimsOnlineServerHostMod/Presentation/`. A useful rule when extending coverage is to identify the actual shared interface or publisher that produces a line, not to infer coverage from the name of a gameplay interaction.

The line format is a pipe-delimited string. Sequence `0` is used by synthesized driver history; live ring events use positive sequence values. `DedicatedServer.Tests/JoinTraceTests.cs` includes this example history and live ordering:

```text
0|A|7|1|1|2|3|0
11|P|7|4|5
12|R|7|9|6|0|1
13|X|7
```

These are test fixtures illustrating the stream parser's order, not a complete public catalog of line kinds. A line's operation and parameter arity must be read from its matching recorder and replay handler before editing a consumer.

## Target selection and polling bounds

`PresentationStream.Targets` starts from claimed Sims, then includes Sims whose `LotId` falls in the interest lots computed by `MirroringInterestScope.ComputeInterestLots`. This keeps recorder traffic scoped to claimed actors and occupied/interesting lots rather than every Sim in the world.

Current controller bounds in `DedicatedServer/Mirroring/PresentationStream.cs`:

| Constant/setting | Value | Effect |
|---|---:|---|
| `SIMSONLINE_PRESENTATION_POLL_MS` | accepted `50..1000` milliseconds | Optional dump-pump interval |
| `PumpInterval` fallback | `250` milliseconds | Used for missing/invalid environment value |
| `MaxDumpPages` | `16` | Maximum dump pages per pump tick |
| `MaxLinesPerEvent` | `200` | Maximum line count in a `SimAnimation` event |
| `ReplayHexBudget` | `3400` hex characters | Default replay-batch budget |
| `FromNowCursor` | `long.MaxValue / 2` | Start a fresh ring cursor at the current end |

`ResolvePumpInterval` accepts only an integer in the stated range; a malformed or out-of-range value falls back to 250ms. The engine mod has a separate push mode with its own constants; do not treat these controller polling constants as universal transport limits.

## Controller and late join

`AuthoritativeSession` requests recorder pages and packages live lines into `SimAnimation` events in chunks of at most `MaxLinesPerEvent`. The stream carries sequence order; line order within a batch and across batches matters for a driver's lifecycle.

A joining player receives history for its targets, rather than a replay of the entire ring. The session gathers the driver's snapshot/history for those targets, holds that player's live lines while gathering, then delivers history before the held updates. Existing clients continue to receive live lines. `DedicatedServer.Tests/JoinTraceTests.cs` tests `AuthoritativeSession.HoldsTheJoinersStreamWhileItsHistoryGathers` and verifies that the joining player gets its history before held live records, while other clients receive live records without waiting for the history gather.

This mechanism is specifically for current driver presentation. It is not a transcript of every action that occurred in the world and should not replay completed transient effects just because they remain in an old buffer.

## Replay batches and client gate

`PresentationStream.EncodeReplay` UTF-8-encodes the joined lines and returns hex. `ReplayBatches` includes newline bytes in its budget calculation and splits the lines so each encoded batch fits the default `ReplayHexBudget`. The bridge handler `HandleSacsReplay` rejects non-client roles, requires a SACS proxy and a payload, decodes the text, and applies the batch through `ApplyReplayBatch`.

Example API-level batching:

```csharp
IReadOnlyList<string> lines = new[] { "11|P|7|4|5", "12|R|7|9|6|0|1" };
string wireText = PresentationStream.EncodeReplay(lines);
IReadOnlyList<IReadOnlyList<string>> batches = PresentationStream.ReplayBatches(lines);
```

The call demonstrates the encoder, not an external command line. The hex budget exists because the bridge frame has a finite payload, and hex doubles the encoded UTF-8 byte count.

## Ordering, state, and history

Driver lifecycle records must be applied in order. A state-like operation may need actor/driver creation before a state transition; a transient operation may not be meaningful after the moment it occurred. The current late-join path uses the driver's own bounded history rather than attempting to replay every historical ring item.

`SacsRecorder` keeps a head/tail history with a reduced representation of settings and states that fell in the gap, and tracks a driver's live state. This is why a late-join snapshot is more than simply â€œsend the last line.â€� Exact order and field semantics are owned by the producer/replay handlers, not by a generic client JSON model.

| Stream issue | Interpretation |
|---|---|
| No line emitted | Producer did not observe the operation, it was excluded, or capture was disabled |
| Line exists but no event | Dump/push/target filtering or controller path issue |
| Event arrives but not applied | Client role, decode, replay handler, or missing proxy issue |
| State appears then regresses | Ordering, driver introduction, local writer, or stale history issue |
| Late join misses an active pose | Driver history/snapshot or target selection did not restore it |

## Tests and source map

- `DedicatedServer/Mirroring/PresentationStream.cs`: parsing, bounds, target selection, encoding, and batching.
- `DedicatedServer/Authority/AuthoritativeSession.cs`: live and join-history orchestration.
- `DedicatedServer/Mirroring/PresentationStream.cs` plus `SimsOnlineServerHostMod/SacsRecorder.cs`: recorder/report conventions and driver history.
- `SimsOnlineServerHostMod/ServerHostMain.cs`: `SACS_RECORDER` and client-only `SACS_REPLAY` bridge handlers.
- `DedicatedServer.Tests/JoinTraceTests.cs`: targeted history and held-live ordering tests.

When changing a record, verify the host producer, parser, event packaging, client handler, and regression test together. A controller unit test cannot prove a game animation or object mapping is visually correct.

Related: [Presentation ownership](presentation-ownership.md), [Input capture pipeline](capture-input-pipeline.md), [Protocol](../protocol.md), [Testing](../testing.md).

## Target selection and audience

The stream does not broadcast every recorder line to every player. `PresentationStream.Targets` begins with claimed Sims and adds Sims whose lot is in the computed interest set. Interest is derived from player claims and occupied/nearby lots; the exact scope is maintained by `MirroringInterestScope`, not by the recorder ring itself.

This means a stream can be healthy while a remote Sim outside the player's interest scope is absent. It also means the event audience is a privacy boundary: expanding the scope can expose unrelated activity and increase traffic. Add target scope with tests that cover both inclusion and exclusion.

The controller's target list is a set of runtime Sim IDs. Client mapping still has to resolve the target in its own world. A target included in the server scope can be skipped on the client if the Sim was not yet spawned or its alias is missing.

## Dump paging and cursor behavior

Each pump requests a bounded number of dump pages. The cursor tracks sequence progress; a fresh `FromNowCursor` starts at the current end to avoid replaying the entire historical ring. Active driver history is gathered separately for late join so a current animation can be reconstructed without replaying all old events.

A stale or incorrect cursor can create duplicate replay or a gap. When diagnosing an event sequence, compare:

1. Recorder sequence at the host.
2. Requested cursor and returned page sequence.
3. Controller's next cursor after the page.
4. Sequence order in `SimAnimation` event batches.
5. Client's bridge batch boundaries and skipped-line counts.

A controller poll interval controls observation latency, not game simulation tick rate. Increasing it may reduce polling but delay presentation; reducing it does not make the game run faster.

## Event sizing and failure behavior

`MaxLinesPerEvent` caps controller packaging; `ReplayHexBudget` caps the encoded client bridge batch; `MaxDumpPages` bounds polling work. These are separate limits at different layers. A large event can be subdivided into replay batches even though it remains one public animation event.

When a payload exceeds a bridge budget, preserve original line order while splitting. Do not truncate a line or silently drop only its suffix, because pipe-delimited operation arguments may be corrupted. If an event is malformed, log an error with the sequence/kind where safe and continue with later valid records rather than freezing the client drain loop.

## Joining midstream

The join history path gathers driver state for the joining player's targets and holds that player's live presentation lines until the history has been sent. This produces the order â€œbaseline/history, then held live updates.â€� Other clients continue to receive live events without waiting for one joiner's history collection.

The history is deliberately bounded to the active driver state. It is not a recording of all interactions since the world started. If the joining client receives a current state but not a transient notification or sound that happened earlier, that can be correct behavior.

## Source of truth for line shapes

A public `SimAnimation` event carries string lines; the public protocol does not enumerate every internal SACS operation as a named JSON record. Each recorder and replay handler owns line arity. Before changing one:

- locate the producer and its exact field ordering;
- locate the parser/consumer and any compatibility behavior;
- check whether the line contains runtime object or driver IDs;
- preserve sequence ordering and delimiter encoding;
- add a positive and malformed-input test;
- check batch size under UTF-8 then hex expansion.

Avoid copying a real recorder line with user-created text or private world identifiers into public docs. Use synthetic values in examples.

## Operational counters and interpretation

The bridge `SACS_RECORDER` command exposes recorder status and ring controls. Status fields are diagnostic; a count proves capture/queueing only at that producer boundary. A replay count on the client proves attempted processing, not visible success. Compare those counters with `MirrorCoordinator` event counts and actual client snapshots to find the first missing transition.

A buildup in `SimAnimation` in-flight work indicates bridge throughput pressure. A steady event count with no applied lines points toward role/decoder/target mapping. A line applied but mismatched visual state points toward local engine context or another writer.

## Sample record flow and validation points

For one active driver, a controller trace should preserve a relationship such as:

```text
host sequence 11: driver acquired
host sequence 12: actor bound
host sequence 13: machine state set
host sequence 14: route/slot update
```

The exact line kinds are recorder-specific; this sequence is illustrative. If line 12 is absent, line 13 may be valid text but meaningless because the client has no actor mapping. If line 14 arrives before line 13, the visual result may differ even though every line was delivered.

Validation should happen at each representation boundary:

| Boundary | Invariant |
|---|---|
| Recorder line | Valid sequence and expected field count |
| Public event | Lines stay ordered and within event line cap |
| Bridge batch | UTF-8/hex encoded bytes stay within budget |
| Client decoder | Malformed lines do not crash the event drain |
| Object mapping | IDs resolve or are safely skipped |
| Replay apply | Client role and SACS proxy are present |
| Visual observation | State appears on intended actor and target |

## Backpressure and latency

A presentation stream has latency from producer observation, page polling, server delivery, client queue delay and bridge application. The fallback poll interval is 250 ms; an environment override is accepted only in the configured 50–1000 ms range. This is a controller polling setting, not a hard end-to-end latency guarantee.

Latency can increase when the bridge is occupied by serial apply commands. Consecutive animation events are batched by `MirrorCoordinator` to reduce round trips, while preserving all raw-line completion accounting. Other event kinds flush the pending animation batch so ordering remains explicit.

When measuring lag, report producer-to-event delay separately from event-to-bridge and bridge-to-visible delay. A single “lag” number hides which queue is accumulating.

## Protocol evolution and compatibility

`SimAnimation` uses string records for the internal SACS boundary because the bridge operation catalog is broader than a small fixed event schema. This flexibility makes producer/consumer version compatibility important. A new line kind must either be safely ignorable by older clients or guarded by negotiated bridge/protocol capability.

Changing a line field order without changing its kind can make older clients parse valid text as different arguments. Prefer a new kind/version discriminator when the shape changes. Add codec fixtures for both current and compatibility forms where old lines remain supported.

## Late join correctness tests

A focused late-join test should arrange a relevant active driver, pause the history gather, emit live lines, and then release the gather. Assert that:

- the joiner receives setup/history first;
- held live lines follow history in increasing sequence order;
- another player receives those live lines without waiting;
- unrelated targets are not included;
- stale transient lines outside active history do not replay.

`JoinTraceTests` already covers ordering and held-live behavior. Extend it when target selection or page/cursor semantics change.

## Line-processing checklist

When a `SimAnimation` event arrives, the client can batch consecutive records to reduce one-command-per-event overhead. The order of lines remains the order established by the host recorder. The replay bridge applies lines in sequence and can skip operations whose referenced actor/object is unavailable. The controller's completion accounting still observes each raw input line so integration tests and in-flight counters are not lost when several events are combined.

A line-specific failure should not stop the public receive queue. The client can report skipped or malformed entries while later events continue. Do not repair a bad line by shifting field indexes or accepting arbitrary extra arguments; strict parsers prevent one unsupported extension from corrupting subsequent records.

## Interest-scope changes

The target set is recomputed from claims and current interest lots. Lot-local ownership can affect which Sims the client simulates, while presentation target selection concerns which drivers' records it receives. These are related but not identical policies. Check both when an owned Sim is absent from a replay event.

A player moving lots can cause a Sim to enter or leave scope. The controller should stop forwarding unrelated history after scope exit and initialize the active state of newly relevant Sims. The stream is scoped to the current view, not a subscription to every actor permanently.

## Player scope and fresh cursors

A fresh presentation cursor deliberately starts at the end of the ring. It avoids flooding a new client with old state, but it means the controller must collect active-driver history through the separate join path. A test that checks only live lines after joining can miss a broken initial pose; a test that checks only driver history can miss ordering with live updates.

A claimed Sim is a target, but related Sims can enter through interest lots. The target set is per player and may differ between players in the same world. Do not copy one player's event list as the expected event list for every client.

## Updating stream limits

The constants solve different bottlenecks: poll interval controls query cadence; dump pages cap host work; lines per event cap public-event size; hex budget caps bridge replay payload. Tune them independently and add tests at boundary values. A larger line count can increase memory and bridge latency even if the public frame remains valid.

When a producer's line grows because of new arguments, revisit both event line caps and encoded bridge budget. A line count test alone does not catch UTF-8/hex expansion.

## Changes that require a live scenario

A change to line batching or parser arity can be established with unit tests. A change to target scope, active driver history or local SACS writer contention needs live-game evidence because process-local animation state and engine scheduling are involved. A change to the public wire DTO should also update the pinned JSON fixture.

When a live visual check is not feasible, state that explicitly and keep the claim limited to decoded/accepted commands. The [scenario matrix](presentation-scenarios.md) provides reusable setups and evidence levels.

## Boundary-specific failure response

- Poll timeout: keep the current cursor and retry within the poll policy; don't fabricate an empty event.
- Too many dump pages: stop after the configured cap and continue on a later pump; avoid monopolizing the host game.
- Too many event lines: split into ordered events rather than truncating.
- Bridge batch too large: split on line boundaries and verify encoded byte budget.
- Missing driver/object: skip only dependent records and report mapping failure.
- Client role mismatch: reject the replay command rather than mutating server gameplay.
- Client disconnected: stop accumulating an unbounded targeted history for that connection.

The recovery action should match whether the data is a current state (request a fresh baseline) or transient effect (do not replay stale work).

## Consumer responsibilities

A client consumer should reject replay when its role is not client, preserve ordering, bound decoded line count/size and tolerate missing objects without aborting unrelated lines. It should report a partial result where possible. A server consumer should never execute client replay records as authoritative gameplay commands.

When a replay event is received after a full snapshot, verify that the event is still current for its driver; a fresh snapshot does not necessarily invalidate a current animation, but a stale buffered setup sequence must not overwrite it. The controller's held-live join ordering and cursor policy define this boundary.

## Field-level comparison

When comparing two processes, align samples by a server tick or event sequence where possible. Record the server/client `ObjectId` mapping and compare stable Sim identity, not raw runtime IDs. A position comparison should use a stated tolerance and exclude moving/routing intervals if the goal is idle drift; an animation-state comparison should name the recorder field and visual observation separately.

Do not sum every mismatch into one score without weighting or categorizing fields. A wrong held object and a different local idle animation have different player impact and different owning subsystems.

Where images are used, keep server/client framing and sample timing aligned; otherwise an apparent animation mismatch may be only a camera/time offset.

A reported visual result should identify its exact source stream and the independent observation used to confirm it.

A live comparison should name the tested build and runtime mode; it is not a blanket compatibility result.

A successful handler result is a transport milestone, not a visual correctness result.
