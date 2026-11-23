# Client game runtime and mirror roles

The “client VM” in this section means the client's own Sims 3 runtime/process, not a virtual machine implemented by Sims3Online. The player runs or attaches to a local game instance; the controller connects that instance to the server through its host bridge. There is no custom bytecode interpreter, full remote-desktop stream, or deterministic clone of the server simulation in this codebase.

This distinction matters because the local game owns its own objects, object ids, simulator tasks, animations, inventories, UI and save files. The network protocol transports selected semantic state and intents; the bridge translates those into operations against the local game.

## Three runtime roles

`ClientMirrorHostFactory` selects one of three modes:

| Mode | Configuration | Effect |
|---|---|---|
| Disabled | `EnableMirror` false | No mirror host or engine |
| Logging | Mirror enabled without local TS3 flags | `LoggingMirrorHost` records/prints dispatches; no game is mutated |
| Client game | Mirror enabled with both `--client-ts3` and `--client-save` | Starts a local TS3 host, sets client role, loads/attaches save, then applies mirror events |

A half-configured client game mode is rejected. This avoids appearing to mirror a live game when one of the essential local host/save arguments is absent.

For client game mode, `SimsOnline.Client.Program` creates the host before connecting the public event receive loop. It starts the bridge, sets `HostRole.Client`, fetches the server world when configured or uses the selected local save, loads the save through the normal game path, and only then begins dispatching incoming events. This readiness order reduces the chance that the first snapshot arrives before the local world can accept it.

## Server and client authority split

The server game runs authoritative interactions, decisions and world changes. The client game is a local presentation/mirror target plus a source of player intent for owned Sims. A client's `HostRole.Client` controls which bridge commands it accepts; it is not permission to decide canonical server state.

| Operation | Server role | Client role |
|---|---|---|
| Run authoritative gameplay interaction | Yes | Not the canonical interaction result |
| Capture supported user intent | Accept/validate | Capture from local UI/input boundary |
| Apply server snapshot positions | Produce snapshot | Apply positions to local copies |
| `SACS_REPLAY` | Record/publish | Apply selected presentation lines |
| `SHOW_DIALOG` | Create request/hold answer | Display request locally |
| `DIALOG_ANSWER` | Apply owner-authorized answer | Not authoritative |
| Native world save | Canonical checkpoint | Local save is a separate process/save |

The role gate is implemented in host bridge handlers, not merely in UI conventions. A bridge command sent to the wrong role should fail rather than mutate the wrong side.

## Process-local object identity

A server `ObjectId` identifies an object in that server process. The client's matching Sim or object may use a different local `ObjectId`. The mirror uses aliases, `SimDescriptionId` where available, spawn/despawn records, and `PresentationApi.ReplayObject` mapping to resolve counterparts. A network ID is not a raw pointer and is not guaranteed to match a local object ID.

When resolution fails, the system may skip that operation or report target-resolution failure. It should not invent a local object based on a number alone. Definition identities such as `bookId` are a different kind of key and have different lifecycle expectations than runtime object ids.

## The local simulator and task context

Some game operations must execute inside a simulator task instead of the controller thread. `SHOW_DIALOG`, for example, schedules `RemoteDialogRunner` with a one-shot simulator task. Other bridge handlers apply object state through local host APIs. A bridge response can mean a command was queued or parsed; it does not always mean the game's asynchronous operation has completed.

This asynchronous boundary creates familiar race conditions:

- A snapshot references a Sim before its local counterpart has spawned.
- A stale alias exists after a local object was removed.
- A local interaction is using an item when a mirror membership change arrives.
- A UI modal pauses local time and must not echo as global speed intent.
- A queued bridge command times out while the game continues to process it.

Each path has its own retry or fallback policy. Do not treat “retry every command” as a universal solution; some actions are non-idempotent.

## Saves and local worlds

`--client-world-from-server` obtains a native `WorldSave` folder and loads it through the game's standard save path. That is an explicit world-bootstrap path. The standard snapshot stream does not update the client's saved DBPF files; neither does SACS animation replay. Conversely, a native world save may load successfully but still differ from the live server until the current snapshot and presentation streams catch up.

`WorldSaveWriter` stages transfer files and promotes the completed folder; see [Save/load pipeline](../data-and-persistence/save-load-pipeline.md). Household export payloads and inventory membership are separate representations and do not replace the client world bootstrap.

A client can also attach to a warm local game process. In that case the server sends a fresh join baseline and presentation history, and local mirror state is reset/repopulated according to the host attach path. A warm attach is not equivalent to loading a fresh transferred world; tests and diagnostics should identify which one was used.

## Observability-only mode

`LoggingMirrorHost` is useful for tests and controller-only runs because it records commands without requiring the game. It proves which commands the controller attempted and in what order. It does not prove:

- that a bridge frame was accepted by Sims 3;
- that the target ID resolved in the client process;
- that a UI displayed;
- that an animation rendered correctly;
- that a state write survived save/reload.

When reporting evidence, name the mode. “Mirror dispatch passed in logging mode” is precise; “the client mirrored the Sim” would overclaim.

## VM versus mirror feature matrix

| Concern | Local runtime owns it | Protocol/mirror responsibility |
|---|---:|---|
| Game simulator/task scheduling | Yes | Schedule supported bridge calls at valid boundaries |
| UI widgets and modal windows | Yes | Carry displayable text/options and answer correlation |
| World object instances | Yes | Alias/map supported identities; never serialize pointers |
| Native save format | Yes | Transfer files or request game-owned save operation |
| Current selected Sim positions | No, server baseline | Deliver snapshot records and apply corrections |
| Driver animation operations | No, server baseline | Record selected SACS calls and replay in order |
| Gameplay outcome | Server authority | Return intent/answer and report accepted/rejected result |

The controller is therefore a coordinator between independent game runtimes, not a virtual-machine monitor that controls their memory or scheduler directly.

## Operational diagnosis

Before investigating a visual discrepancy, record:

- whether mirroring is disabled, logging-only or connected to a client TS3;
- client process role and loaded save name;
- whether the process was freshly loaded or warm-attached;
- whether the client fetched a server `WorldSave`;
- which server/client build and host mod stamp are active;
- whether the relevant object alias is known;
- whether the discrepancy concerns snapshot, interaction replay, SACS presentation, dialog UI or inventory membership.

Do not include save contents, credentials, private paths or real user-created household exports in public bug reports. A small synthetic protocol fixture is normally sufficient to reproduce a controller/codec defect.

## Tests and source map

- `DedicatedServer/Mirroring/ClientMirrorHostFactory.cs` — mode selection and config validation.
- `SimsOnline.Client/Program.cs` — host startup, role set, world load/attach and receive flow.
- `DedicatedServer/Authority/HeadlessSims3Host.cs` — host bridge lifecycle and local game commands.
- `DedicatedServer/Mirroring/LoggingMirrorHost.cs` — observability-only adapter.
- `DedicatedServer/HostBridge/BridgeLineCodec.cs` — bridge command wire shapes.
- `SimsOnlineServerHostMod/ServerHostMain.cs` — host-role command registration and role checks.

Factory tests verify configuration branches. Bridge/host tests validate controller commands and role gates. A live client runtime is needed to verify actual UI, object resolution and visual presentation. See [Client mirroring](client-mirroring.md) and [Presentation scenarios](presentation-scenarios.md).

## Startup sequencing and reconnects

The client controller starts its receive queue before the join handshake so it can observe join responses without losing lines. It drains `JoinSession` replies and the initial baseline before arming input capture. This avoids treating setup writes, such as a baseline speed or position apply, as new player intent.

After the join baseline, the controller sends `SetClientRole`, may send a claim request, and then enters the event loop. The server's role/claim events update the local authority gate before the next relevant events are mirrored. A reconnect or warm attach can receive late-join history again; the client mod must reset its local mirror/history state so duplicate old drivers do not accumulate.

A startup failure before session connect should leave the local game process cleaned up by the outer lifecycle. A failure during receive should stop the reader before teardown bridge writes. These lifecycle boundaries are why process start, bridge readiness, world load and public TCP join are separate observable phases.

## Safe and unsafe local writes

A mirror write should be idempotent or tied to a unique event/command identity. Applying a position repeatedly is usually a correction; re-queueing a non-idempotent interaction can create duplicates. `InteractionQueued` self-echo filtering and `ActiveOrderTracker` position stripping address two different double-writer hazards.

Examples:

| Local action | Retry risk |
|---|---|
| Set a Sim position to latest snapshot | Usually safe to reapply, but can interrupt routing |
| Apply autonomy suppression | Must balance enable/disable lifecycle |
| Queue an interaction | May duplicate order unless deduped/coalesced |
| Replay a one-shot sound/VFX | Duplicate audible/visual effect |
| Show a dialog | Duplicate prompt or stale answer correlation |
| Add/remove inventory member | Can conflict with in-use objects |

Do not add generic retry loops without classifying the command's idempotence and identity semantics.

## Mode-aware diagnosis

`MirrorEngine.InteractionReplayEnabled` and `LotLocalAuthorityMode` affect filtering and translation. In interaction-replay-off presentation mode, the client can mirror server positions for owned Sims as well as remote Sims; in client-native mode, local ownership/autonomy rules and active orders can strip selected positions. A report that omits these modes can describe expected behavior as a defect.

A snapshot arriving at the controller may be filtered differently for two clients due to ownership, active orders and interest scope. Compare the raw server snapshot with the per-client filtered event before comparing local game state.

## Startup sequencing and reconnects

The client controller starts its receive queue before the join handshake so it can observe join responses without losing lines. It drains `JoinSession` replies and the initial baseline before arming input capture. This avoids treating setup writes, such as a baseline speed or position apply, as new player intent.

After the join baseline, the controller sends `SetClientRole`, may send a claim request, and then enters the event loop. The server's role/claim events update the local authority gate before the next relevant events are mirrored. A reconnect or warm attach can receive late-join history again; the client mod must reset its local mirror/history state so duplicate old drivers do not accumulate.

A startup failure before session connect should leave the local game process cleaned up by the outer lifecycle. A failure during receive should stop the reader before teardown bridge writes. These lifecycle boundaries are why process start, bridge readiness, world load and public TCP join are separate observable phases.

## Safe and unsafe local writes

A mirror write should be idempotent or tied to a unique event/command identity. Applying a position repeatedly is usually a correction; re-queueing a non-idempotent interaction can create duplicates. `InteractionQueued` self-echo filtering and `ActiveOrderTracker` position stripping address two different double-writer hazards.

Examples:

| Local action | Retry risk |
|---|---|
| Set a Sim position to latest snapshot | Usually safe to reapply, but can interrupt routing |
| Apply autonomy suppression | Must balance enable/disable lifecycle |
| Queue an interaction | May duplicate order unless deduped/coalesced |
| Replay a one-shot sound/VFX | Duplicate audible/visual effect |
| Show a dialog | Duplicate prompt or stale answer correlation |
| Add/remove inventory member | Can conflict with in-use objects |

Do not add generic retry loops without classifying the command's idempotence and identity semantics.

## Mode-aware diagnosis

`MirrorEngine.InteractionReplayEnabled` and `LotLocalAuthorityMode` affect filtering and translation. In interaction-replay-off presentation mode, the client can mirror server positions for owned Sims as well as remote Sims; in client-native mode, local ownership/autonomy rules and active orders can strip selected positions. A report that omits these modes can describe expected behavior as a defect.

A snapshot arriving at the controller may be filtered differently for two clients due to ownership, active orders and interest scope. Compare the raw server snapshot with the per-client filtered event before comparing local game state.

## Configuration and factory tests

`ClientMirrorHostFactory` tests cover the disabled mode, default logging mode, client headless mode, both half-configurations and session option propagation. These tests are valuable because configuration mistakes otherwise look like a network or game failure: the controller can receive events while no local game host exists.

The factory returns both a logging host and, when configured, an owned client host. The `MirrorEngine` is created only when a host adapter is available. Tests should verify the selected host type and not infer game process behavior from the factory result.

## Lifecycle and failure matrix

| Lifecycle phase | Expected local state | Failure response |
|---|---|---|
| Controller start | No host until factory selects a mode | Reject malformed/half client-game configuration |
| Bridge start | Local game mod handshake available | Fail start before public event dispatch |
| Role set | Host accepts client/server role | Gate subsequent commands by the selected role |
| Save load | Game reaches world-ready state | Do not treat process start as loaded world |
| Public join | Join response and baseline drained | Don't arm input capture before baseline |
| Event receive | Socket reader drains into ordered queue | Stop reader before teardown bridge writes |
| Warm attach | Existing process identity preserved | Reinitialize mirror state and receive join history |
| Leave | Best-effort lot export, then session leave | Do not block shutdown on export failure |

A client can have a running process and a connected bridge while no public session exists. It can also be connected to the public session in logging-only mode without a game process. Report those as separate status facts.

## Telemetry boundaries

The client controller logs host startup, bridge readiness, session join, role request, claim request, bridge trace and mirror dispatch. Those are operational breadcrumbs, not private account identifiers. The local process id and build stamp are useful for correlating the host but should not be confused with session/player identifiers.

For privacy, keep `--bridge-record` outputs local: frame records may include arbitrary text arguments, identifiers and player content. Use a synthetic command/fixture when filing a reproducible wire bug.
