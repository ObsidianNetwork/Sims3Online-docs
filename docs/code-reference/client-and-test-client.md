# Client and test client

The client-side code is split into a .NET controller and a minimal protocol exerciser.
`SimsOnline.Client` is the player-facing controller process: it connects to the dedicated
server, manages the public wire, receives authoritative events, and dispatches state to the
local game bridge when configured. `SimsOnline.TestClient` is a small CLI used to exercise
the server listener without running the full local mirror path.

Neither executable is the authoritative multiplayer simulation. The server controller and
its host game own authoritative session policy; the client submits commands and presents
server results. For shared message definitions, see
[protocol-library.md](protocol-library.md).

## `SimsOnline.Client` responsibilities

`SimsOnline.Client/Program.cs` coordinates configuration, network, bridge, and mirroring.
The source documents the main CLI options and their defaults:

| Option | Current default / behavior |
|---|---|
| `--endpoint <ip:port>` | `127.0.0.1:34019` |
| `--player <int>` | `1` |
| `--display <name>` | `client-scaffold` |
| `--role <server|client>` | `client` |
| `--connect-timeout <seconds>` | `30` |
| `--max-events <n>` | `-1` means run indefinitely; nonnegative value exits after N events. |
| `--exit-after-role-changed` | Exit after the role-change event; used by integration tests. |
| `--exit-after-mirror-applied <n>` | Exit after N mirror apply operations. |
| `--enable-mirror` | Create `LoggingMirrorHost` and `MirrorEngine` and dispatch incoming `ServerEvent`s through the mirror path. |
| `--json-stdout` | Emit machine-readable JSON lines rather than human-readable log text. |
| `--snapshot-log <path>` | Divert complete `full_snapshot` / `snapshot_delta` receive lines to a separate file and keep a compact main-log stub. |

Additional options in the parser configure client-side game lifecycle, state/log output,
bridge tracing, authority gate, claim handling, and world-from-server transfer. Verify
argument parsing in `Program.cs` before scripting undocumented flags; the summary above is
not a substitute for the full parser.

The controller accepts `--wire json` or `--wire line`. JSON is the default in the test
client; the client chooses its mode from its arguments and uses `PublicWireConnection` to
translate line-v1 traffic back into the existing JSON event path. This keeps mirror dispatch
independent of which public wire encoding the socket uses.

### Connect and join sequence

`Program` creates the TCP connection with a timeout, reads server lines, optionally performs
`PublicWireConnection.HandshakeAsync`, and maintains a command/event loop. A join reply
helper (`JoinReplies`) keeps track of the join response and the state/capabilities needed
before continuing. When a role is declared, `SetClientRole` is sent after the join; an
optional Sim claim follows that role declaration.

The client may request the server's world before it loads a local save. `RequestWorld` is a
distinct command and does not require `JoinSession`; the program validates that the
configured local save directory is the client game's Saves directory when world-from-server
transfer is enabled. Received `WorldFileChunk` records are assembled by `WorldSaveWriter` in
the shared protocol library.

The command/action path is asynchronous and command-id aware. A response for one issued
command is matched by its `CommandId`; unrelated broadcasts are not treated as that
command's answer. `LeaveSession` is sent during graceful shutdown except for tests/flows
that intentionally exit after a role event.

## Public line adapter: `PublicWireConnection`

`SimsOnline.Client/PublicWireConnection.cs` adapts one line-v1 socket to the controller's
internal JSON line processing. It owns per-connection command ids, correlation ids, resume
state, bulk assembly, event sequence cursor, and pending deliveries.

| State / limit | Purpose |
|---|---|
| `CommandIds` | Maps full JSON `Guid` command IDs to connection-local `ulong` ids without truncation. Retries reuse their mapping. |
| `correlations` / `commands` | Associates line-v1 correlation sequence with command GUID for matching acknowledgements. |
| `Epoch`, `ResumeToken`, `LastDeliveredSeq` | Credentials/cursor for warm reconnect. |
| `pending`, `ready`, `eventOwners`, `eventBulkOwners`, `ackOwners` | Keep event/ack payloads ordered while associated bulk content is incomplete. |
| `MaxBulkBytes` | 256 MiB assembly bound. |
| `MaxInFlightBulks` | 16 concurrent bulk assemblies. |
| `MaxPendingEvents` | 1024 pending events. |

A successful warm resume preserves connection identity and event cursor. A fresh resume
response clears the command/correlation maps, resets bulk and delivery state, and starts
from sequence zero for the new epoch. The handshake checks protocol version, epoch
consistency, and the expected first replay sequence. Invalid resume credentials fail rather
than being accepted partially.

`PublicWireConnection.ValidateMode` accepts only `json` or `line`. The socket write path
serializes command encoding with the write so a correlation map update cannot race a
concurrent write. Completed line events are turned into the same JSON event representation
used by the normal client loop. JSON sockets bypass the line adapter entirely.

## Receive queue and single-owner dispatch

`SimsOnline.Client/ReceiveQueue.cs` separates socket reading from event handling. The socket
reader writes `(Line, timestamp)` entries to a channel; handlers consume lines from its
`ChannelReader`. The queue exposes `Depth`, `OldestAge`, and `ReceiveLag` information,
preserving source order while making it possible to diagnose handlers that are behind. The
reader can continue draining the socket while an event handler awaits slower bridge work.

That separation is deliberate: directly running host bridge operations in the socket read
loop would stop network intake whenever a mirror application stalls. The queue is unbounded
in the current client design, so it avoids dropping deltas at ingress but can accumulate
backlog if consumption remains slower than arrival. Lag reporting is the visibility
mechanism; it is not a memory cap.

`SimsOnline.Client/MirrorCoordinator.cs` owns the drain-loop mutation path. Its important
operations are:

* `Post(write, work)` schedules timer or background work onto the drain loop, preserving
  event order.
* `ReadLineAsync` is used by join-drain logic that reads before the main run loop; it also
  services posted work in sequence.
* `Merge(deltas)` folds a batch of snapshots using latest-tick-wins per complete
  `SimSnapshot`/`ObjectSnapshot` record. It does not merge individual fields.
* `RunAsync` drains available lines, batches snapshot deltas up to the configured per-cycle
  fold limit, dispatches non-snapshot events in order, and handles forced-full/resync
  behavior.
* `CoordinatorConfig` controls the maximum fold batch; the session config passes an
  effective default of 8.

Interaction events have specialized batch dispatch so bridge round-trips can be combined
without changing event type order. Other event families are not coalesced across types.
Drift is detected on raw delta tick-contiguity, not on the merged endpoint. One
snapshot-request arbiter debounces client resync requests; an in-flight server-forced full
snapshot suppresses redundant client requests.

`CoordinatorTimers<TKey>` schedules/cancels keyed delayed work and posts elapsed callbacks
to the same loop. `OrderCompletionReassert` builds a final snapshot update after a tracked
user order completes; it avoids losing the final position when intermediate position
mirroring is suppressed during the active order window.

### Why whole-record coalescing matters

Suppose a burst contains a Sim position record at tick 10 and a different record for the
same Sim at tick 11. `Merge` takes the entire tick-11 `SimSnapshot`, not a tick-11 position
plus tick-10 metadata. This is safe because each snapshot row is treated as one coherent
record. Coalescing individual fields could combine stale ownership, posture, queue, and
position information from different authoritative ticks.

Non-snapshot events such as interaction lifecycle, claims, autonomy changes, and speed
changes remain ordered. A newer position snapshot cannot replace a command acceptance or
interaction completion event.

## Local host / mirror integration

The client controller can run without an enabled mirror. With `--enable-mirror`,
`LoggingMirrorHost` and `MirrorEngine` handle server events and translate mirrorable events
into host bridge commands. The server's public `PlayerRole` and the local host bridge role
are separate; the local bridge gate has `Off`, `Observe`, and `Enforce` modes.

The client loop also handles:

* snapshot full/delta dispatch and `AckSnapshot` after application;
* event translation and filters that prevent client-owned or intentionally hidden Sims from
  being mirrored as server puppets;
* world file writes and client save loading when the server world is requested;
* optional remote dialogs and missing-description requests;
* graceful claim/session cleanup before the connection closes;
* machine-readable logs, snapshot log diversion, and optional telemetry context.

For mirroring policy source, read `DedicatedServer/Mirroring/MirrorEngine.cs`,
`MirrorEventFilter.cs`, and `ClientMirrorHostFactory.cs`. The client executable owns
orchestration; policy helpers live in the server assembly so the dedicated server and client
do not duplicate conversion rules.

## `SimsOnline.TestClient`

`SimsOnline.TestClient/Program.cs` is a single-file console client. It supports both `json`
and `line` wire modes and defaults to the server's local development endpoint
`127.0.0.1:34019`. The source documents these defaults:

| Option | Default / purpose |
|---|---|
| `--endpoint` | `127.0.0.1:34019` |
| `--wire` | `json`; also accepts `line`. |
| `--player` | `1`. |
| `--display` | `probe`. |
| `--connect-timeout-seconds` | `30`. |
| `--read-timeout-seconds` | `30`. |
| `--claim-sim <id>` | Optional ownership lifecycle smoke test. |
| `--rival-sim <id>` | Optional cross-ownership rejection test. |
| `--skip-snapshot` | Skip snapshot request in ownership cases. |
| `--queue-x`, `--queue-y`, `--queue-z` | Optional target position for `Terrain.GoHere` exercise. |
| `--interaction-wait-seconds` | `60`. |
| `--delta-snapshot` | Compare full and delta snapshots in a paused world. |
| `--delta-wait-seconds` | `5`. |

Default mode performs `JoinSession`, `RequestSnapshot`, and `LeaveSession`. `--claim-sim N`
exercises claim, idempotent claim, release, reclaim, and auto-release on leave. Combining
`--claim-sim` with coordinates queues a speed change and `Terrain.GoHere`. Adding
`--rival-sim M` attempts a second actor and expects rejection. `--delta-snapshot` is
mutually exclusive with the claim flow.

The reader waits for `CommandAccepted` or `CommandRejected` matching the command id it sent.
It can print unrelated broadcast events without mistaking them for the request's response.
This is important in a multi-client test: “some response arrived” is not equivalent to “this
command was accepted.”

Example smoke invocation (replace the sample object id with one present in the loaded
world):

```powershell
dotnet run --project SimsOnline.TestClient/SimsOnline.TestClient.csproj -- --endpoint 127.0.0.1:34019 --player 1 --display smoke --claim-sim 100 --queue-x 4 --queue-y 0 --queue-z 9
```

## Failure modes to distinguish

| Symptom | Likely boundary |
|---|---|
| Connect timeout/refusal | Endpoint/listener/process startup, not JSON decode. The client retries only where its connection path explicitly implements retry. |
| `WELCOME` unsupported or malformed | Line-v1 handshake/version/frame decode. |
| Join rejected | Session/identity/role/capability policy; check `SessionJoined`/`CommandRejected` reason. |
| Client receives but mirror does not apply | `MirrorCoordinator` dispatch, filters, `MirrorEngine`, or local bridge; inspect the original event and subsequent command. |
| Increasing receive depth/oldest age | Handler or local host work is slower than socket intake; queue reports expose lag but do not cap backlog. |
| Resume starts fresh | No valid cursor/token pair, epoch mismatch, or server replay window unavailable. Fresh state resets local sequence/correlation state. |
| Snapshot not acknowledged | It was not applied, application was interrupted, or the ACK uses the wrong tick. Do not ACK on receipt alone. |
| World transfer fails | `WorldSaveWriter` rejected path/name/order/offset or a pre-existing unmarked save folder. Existing user save should remain intact. |
| TestClient waits forever | It only accepts command-id-matching result; verify the server replies with the issued id and check read timeout. |

## Tests and next files

Client-focused cases are in `ClientReceiveDrainTests.cs`, `ClientWireLineTests.cs`,
`CoordinatorTimerTests.cs`, and `OutboundListenerTests.cs`. The line suite covers resume
cursor and identity, malformed line handling, bulk assembly, join/snapshot/delta, world
fetch, and test-client smoke. The receive suite covers order, reader errors, handler stalls,
and lag reporting.

Use a case-name filter, for example:

```powershell
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter SimsOnlineClient
```

For the listener and authorization side, read [dedicated-server.md](dedicated-server.md).
For exact wire serialization rules, read [protocol-library.md](protocol-library.md); for the
matching host-side filters and hooks, see [server-host-mod.md](server-host-mod.md).

## Ordering detail for event-loop changes

The public socket, client receive queue, mirror coordinator, and local host bridge form a
chain, but they do not share one lock or one thread. Use the existing message handoff
points:

| Handoff | Ordering guarantee / caveat |
|---|---|
| `PublicWireConnection.ReadJsonAsync` | Reassembles line-v1 envelopes and yields complete internal event lines in sequence order. |
| Socket reader → `ReceiveQueue.TryWrite` | Appends a line and receive timestamp; the socket reader is not waiting for mirror completion. |
| `MirrorCoordinator.Post` | Places a work marker in the same drain queue; work runs after lines already ahead of it. |
| `MirrorCoordinator.RunAsync` | Greedily drains available lines up to `MaxFoldPerCycle`; raw event ordering still matters. |
| `tryMirrorDispatch` callback | Central dispatch path applies filters and sends mirror operations. Keep events routed through it. |
| `CoordinatorTimers` callback | Timer completion is posted to the drain loop; cancel/replacement should prevent stale work. |
| `AckSnapshot` send | Acknowledges the applied snapshot tick, not the last received line. |

A timer callback must not mutate mirror state directly from a timer thread. Post it, and let
the owner loop apply it between received lines. This gives the loop one serialization point
for both network messages and delayed work.

`RunAsync` has a single-line fast path and a multi-line batch path. When there is only one
item, it may read ahead for an interaction batch; otherwise it processes a batch by
coalescing snapshots and passing other event types through in order. The `afterLine`
callback is used by controlled integration-test exits and sees the raw line after dispatch.
Tests should verify the final state and event count rather than relying on timing sleeps.

## Interpreting client progress

The client has several different progress indicators:

| Indicator | What it proves | What it does not prove |
|---|---|---|
| TCP connected | The socket connected to the listener. | Join succeeded or a game host is ready. |
| `WELCOME` received | Line-v1 handshake accepted. | The client has joined or loaded the server world. |
| `CommandAccepted` | The session accepted that command id. | Every broadcast side effect has already been applied locally. |
| Event sequence delivered | The wire adapter delivered a complete event envelope. | `MirrorEngine` finished applying it. |
| Snapshot tick acknowledged | Client reports that it applied through that tick. | It has no later events queued. |
| Mirror applied-count exit flag | The requested number of host apply operations occurred. | Every event type was applied or the world is fully converged. |
| Queue depth/oldest age | Receive backlog at the client queue. | Whether the server is late, local bridge is slow, or mirror handler itself is blocked. |

When investigating apparent staleness, capture the socket/event sequence, receive depth and
oldest age, mirror callback outcome, host bridge response, and last `AckSnapshot` tick. One
log line alone cannot identify which handoff stopped progressing.

## Client test cases by transition

`ClientWireLineTests.cs` includes cases for each major line transition: warm cursor/command
identity, typed bounded resume frames, codec failure while preserving drained events,
acknowledgement bulk order/duplicate naming, handshake/join/bulk/delta/animation, world
fetch and raw-bulk assembly, GUID identity in replies, and test-client smoke.
`ClientReceiveDrainTests.cs` covers socket draining while a handler stalls, order/error
propagation, and lag warnings. `CoordinatorTimerTests.cs` covers posted-work order and
canceled/replaced timers. These tests are valuable when modifying queue or sequencing code
because they exercise timing boundaries without requiring a live host game.

The repository's [testing guide](../testing.md) gives the full development runner. For
server-side event enqueue and listener behavior, use
[dedicated-server.md](dedicated-server.md); for framing fields and event schemas, use
[protocol-library.md](protocol-library.md).
