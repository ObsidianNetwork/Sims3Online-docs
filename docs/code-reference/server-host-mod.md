# Server host mod: hooks and interoperability

`SimsOnlineServerHostMod/` is the C# script-mod code loaded by a Sims 3 process. The
repository's README describes it as experimental and requires a compatible, legally acquired
game installation. This page documents only the public integration boundary and this
repository's own types, commands, and files. It does not reproduce game implementation code
or private engine internals.

The controller and mod are separate processes. The controller sends bridge commands; the mod
dispatches them, performs the requested operation on the game thread/host, and writes
correlated replies or events. The mod also records selected state and presentation calls so
clients can mirror the authoritative host's decisions. The mod is not the public TCP
endpoint.

## Boundary layers

| Layer | Project file / source | Responsibility |
|---|---|---|
| Build/package input | `SimsOnlineServerHostMod/build-server-host-mod.ps1`, `package-server-host-mod.ps1`, `Sims3PackageBuilder.cs` | Compile and package mod source for the game environment. |
| Bootstrap | `Initializer.cs`, `ServerHostMain.cs` | Register/initialize the mod and enter its bridge command/pump loop. |
| Bridge framing | `BridgeCodec.cs`, `BridgeMemoryBuffer.cs`, `BridgeSharedRegionLayout.cs`, `BridgeCrc32.cs` | Parse/emit command and event lines, share layout and bulk framing. |
| Command dispatch | `ServerHostMain.cs`, `Handlers/HandlerSet.cs`, `EmbeddedHandlerAssembly.cs` | Route named bridge operations; load selected handler code through a narrow adapter. |
| Recording | `WorldRecorder.cs`, `SimulatorRecorder.cs`, `RouteRecorder.cs`, other `*Recorder.cs` files | Emit snapshots, state, activity, and presentation observations. |
| Authority and filtering | `AutonomyCommandGate.cs`, `OwnedHostDetourFilter.cs`, `SelectionGuard.cs`, emit/capture filters | Gate or filter client-side effects and host-origin messages. |
| Presentation | `Presentation/`, `PresentationModules.cs`, `BoundaryReplay.cs` | Apply streamed presentation to a client process without turning it into an independent simulator. |
| Optional native seam | `SimLinkNative.cs` | Access optional native ASI exports when detected and compatible. |

## Role and authority are distinct controls

The names are similar across the public protocol and local host bridge, but they describe
different boundaries:

| Control | Values | Scope / purpose |
|---|---|---|
| Public `PlayerRole` | `Server`, `Client` | Per connected player in `SimsOnline.Protocol/ClientCommand.cs`. A client declares a role after joining. |
| Bridge `HostRole` | `Server = 0`, `Client = 1` | Role of the local game process in `DedicatedServer/HostBridge/BridgeCommand.cs`; carried by `SET_ROLE`. |
| Bridge `AuthorityGateMode` | `Off`, `Observe`, `Enforce` | Local gate policy sent with `SET_AUTHORITY` plus owned Sim ids. |
| Presenter behavior | No separate `HostRole` enum value | A client process may receive authoritative state and present it. It is an application mode, not a third serialized role. |

The distinction prevents a common mistake: a public client role declaration is not by itself
a local host hook mode, and a client host role does not grant authority over server state.
The server controller validates player identity and ownership before producing bridge
operations.

`BridgeCodec` and `BridgeLineCodec` include commands for `SET_ROLE`, `SET_AUTHORITY`, and
`SET_SIM_SCOPE`. The latter describes server simulation scope; the authority command
describes client-side gate mode and the set of owned Sims. The `AuthorityGateMode` values
provide a gradual enforcement boundary: off, observe, enforce. When troubleshooting a
rejected operation, inspect the public session check and the mod-side role/gate separately.

## The bridge entry and command vocabulary

`ServerHostMain` builds a command-to-handler table. The current table includes lifecycle
operations (`CONNECT`, `DISCONNECT`, `LOAD_SAVE`, `LOAD_SAVE_MENU`, `SAVE_GAME`),
synchronization (`REQUEST_SNAPSHOT`, `BULK_REF`, `SET_GAME_SPEED`, `SET_WORLD_TIME`),
roles/scope (`SET_ROLE`, `SET_SIM_SCOPE`, `SET_AUTHORITY`), interaction/dialog operations
(`QUEUE_INTERACTION`, `CANCEL_INTERACTION`, `LEAVE_CONVERSATION`, `DIALOG_ANSWER`,
`SHOW_DIALOG`), state inspection/application (`PROBE_SIM_STATE`, `APPLY_SIM_STATE`,
`PROBE_SIM_DETAILS`, `APPLY_SIM_DETAILS`), and diagnostic commands prefixed by `DIAG_` or
`PROBE_`.

The bridge command set also includes spawn/despawn and Sim alias operations, lot
merge/export operations, autonomy gate operations, and replay/presentation commands. The
controller's authoritative adapter declares the typed command records in
`DedicatedServer/HostBridge/BridgeCommand.cs`; its line codec is the counterpart to the
mod's `BridgeCodec`. A change to a command name, correlation field, argument ordering, or
escaping must be applied compatibly to both endpoints.

### Handler assembly boundary

`Handlers/HandlerSet.cs` exposes static `Names()` and `Dispatch(name, parts, ctx)` methods.
`Names()` declares the handler names served by this set; `Dispatch` receives the decoded
command parts and a boxed service adapter. The current set is focused on status, position,
motives, cash, skills, autonomy, and queue probes:

| Handler verb | Responsibility at boundary |
|---|---|
| `PROBE_SIM_STATUS` | Read a compact status result. |
| `APPLY_POSITION`, `APPLY_POSITIONS` | Apply one or a batch of positions on the local host. |
| `APPLY_MOTIVE`, `PROBE_SIM_MOTIVE` | Apply/read a motive value. |
| `APPLY_CASH`, `PROBE_HOUSEHOLD_CASH` | Apply/read household funds. |
| `APPLY_SKILL`, `PROBE_SIM_SKILL` | Apply/read a skill level. |
| `APPLY_AUTONOMY`, `PROBE_SIM_AUTONOMY` | Apply/read autonomy state. |
| `PROBE_SIM_INTERACTION_QUEUE` | Inspect interaction queue state. |

The host shell owns the bridge reply and service verbs; the handler returns a small status
value and calls the adapter for results. The engine-coupled implementation is not part of
this public interoperability description. Error paths must be returned through the bridge
response contract rather than inferred from console output.

## Authority/presenter behavior from the outside

A one-simulator session lets the server's game process own gameplay decisions. A client gets
state and presentation updates and submits user intent through the controller. The mod
contains the boundary mechanisms that make this possible:

* `AutonomyCommandGate` is a small state machine with `Register`, `Observe`, `Clear`,
  `IsTracked`, `IsGated`, and `TrackedSims`. Registration records whether a user-directed
  command was already present; later observations decide whether a gate action is needed.
* `ClientCaptureFilter.ShouldForwardCapturedInteraction` is a pure predicate used by the
  capture path; it selects which queued interactions are forwarded as user intent.
* `UnclaimedRoutingEmitFilter` filters routing observations by membership, rather than
  forwarding every observed route.
* `OwnedHostDetourFilter` and `SelectionGuard` constrain which host operations are captured
  or replayed and prevent the replay path from taking over selection.
* `InteractionReplayApplySorter`, `TerminalInteractionFilter`, `RoutingInteractionFilter`,
  and `SitTransitionEmitFilter` classify or order interaction/presentation records before
  application.
* `PresentationModules` provides dispatch delegates for presentation line, reset, report,
  tick, and walk-state callbacks. The individual modules in `Presentation/` implement
  focused handlers.

The server-side public mirror is independently scoped.
`DedicatedServer/Mirroring/ClientAuthorityPolicy.cs`, `MirroringInterestScope.cs`,
`SimulationScopePolicy.cs`, and `AuthoritativeSession` determine which state goes to which
peer. Local hook gating does not replace those controller checks.

## Recording and presentation interfaces

The mod's own recording types correspond to separate observable categories:

| Source family | Observable responsibility |
|---|---|
| `WorldRecorder.cs`, `WorldRecorderRoadPlace.cs` | World-level state and road-place request observations. |
| `SimulatorRecorder.cs`, `ObjectsRecorder.cs` | Sim/object enumeration and snapshot records. |
| `RouteRecorder.cs`, `RouteLegIndex.cs`, `WalkerCopyLine.cs`, `WalkHaltTiming.cs` | Route/walk-related observations and metadata. |
| `AnimationRecorder.cs`, `AudioRecorder.cs`, `LookAtRecorder.cs`, `VfxRecorder.cs` | Presentation signals emitted by the host. |
| `SlotsRecorder.cs`, `InventorySet.cs`, `ObjectUse.cs`, `RelationshipState.cs`, `ConversationState.cs` | Selected object/Sim state in wire-compatible forms. |
| `EventQueueRecorder.cs`, `SacsRecorder.cs`, `SacsPicks.cs`, `SacsRandom.cs` | Event or native-side observations used by controller/dev scenarios. |
| `SimPresenceWire.cs`, `SimDetailsWire.cs`, `SimStateWire.cs`, `SimScopeWire.cs`, `UserIntentWire.cs`, `SimAliasWire.cs` | Compact wire DTO and formatting helpers for host observations/commands. |
| `Snapshot2Writer.cs` | Binary snapshot writer paired with `Snapshot2BinaryReader`. |
| `BridgeBulkBuffer.cs` | Large payload slots and checksum handling. |

These recorders do not make a client authoritative. They describe what happened on the
server or carry a client-originated intent/state bundle through a server-validated path. An
incomplete recorder event is not equivalent to a state mutation; consumers should follow the
corresponding bridge event and public event mapping.

## Tunables and build-time options

`Tunables` is a runtime tuning registry for numeric policy values. `Get(name, fallback)`
registers the fallback and returns a set value if present. `Set(name, value)` accepts an
invariant-culture number; empty or `default` removes the override; malformed values return
an error. Each successful set increments `Version` and synchronizes recorder/native
switches. Values are not persisted: a game restart returns to code defaults.

The mod exposes a command path shown in source as `SACS_RECORDER tune [name value]`. Example
tunable names and defaults in the current `Presentation/Walks.cs` include `mirror.deadband`
(`0.25`), `mirror.posture-snap` (`1.5`), `walk.end-snap` (`0.1`), and `walk.path-goal`
(`0.5`). These values are internal tuning points, not a stable external config schema;
confirm the current source default before relying on one.

`SIMSONLINE_METHOD_SWAP=1` is read by `Sims3PackageBuilder` during package creation; it sets
the initializer tunable that the mod checks before activating the method-swap hook. This is
a build/package-time opt-in, not a runtime environment lookup inside the game. Other broad
behavior switches are sent through bridge commands or `Tunables.Set`.

## File guide: bootstrap and bridge core

| File | Focus |
|---|---|
| `Initializer.cs` | Mod assembly initializer and optional method-swap tunable. |
| `ServerHostMain.cs` | Bridge command map, request handling, host pump and event publication. |
| `MethodSwap.cs` | Guarded method replacement entry point. |
| `BridgeCodec.cs` | Bridge command/event line contract and helper encoders. |
| `BridgeMemoryBuffer.cs` | Shared host/controller memory buffer and control flow. |
| `BridgeSharedRegionLayout.cs` | Region offsets and marker bytes. |
| `BridgeCrc32.cs` | CRC helper shared with the controller implementation. |
| `BridgeBulkBuffer.cs` | Bulk-payload slot protocol. |
| `EmbeddedHandlerAssembly.cs` | Loads and dispatches the optional handler assembly. |
| `EmbeddedLoadTestAssembly.cs` | Loads isolated load-test code. |
| `BuildStamp.cs` | Mod build identifier sent over the bridge. |
| `Sims3PackageBuilder.cs` | Package construction, including optional build-time method swap. |

## File guide: policy, filters, and replay state

| File(s) | Responsibility |
|---|---|
| `AutonomyCommandGate.cs` | Pure command/autonomy gate state machine. |
| `ClientCaptureFilter.cs` | Pure allow/deny predicate for captured user interaction. |
| `UnclaimedRoutingEmitFilter.cs` | Pure membership check for emitted routing observations. |
| `OwnedHostDetourFilter.cs` | Determines whether a host hook is within capture scope. |
| `SelectionGuard.cs` | Keeps replay/presentation from changing selection. |
| `RoutingInteractionFilter.cs`, `TerminalInteractionFilter.cs`, `SitTransitionEmitFilter.cs` | Classify/filter bridge interaction events. |
| `InteractionReplayApplySorter.cs` | Orders replay application operations. |
| `ReplayStateCache.cs`, `ReplaySetupCache.cs`, `StateCatchUp.cs`, `IdleRepairPlan.cs`, `RepairEntryGate.cs`, `RepairLabel.cs` | Cache and reconcile state needed by presentation repairs. |
| `BoundaryState.cs`, `BoundaryReplay.cs`, `DriverIntroduction.cs` | Track boundary recording/replay metadata. |
| `QueueMirrorItem.cs`, `ParamAuthority.cs` | Queue and parameter-authority value helpers. |

## File guide: presentation and data records

`Presentation/Module.cs` is the module-level seam. `Presentation/Walks.cs`,
`Inventories.cs`, `Uses.cs`, `Moodlets.cs`, `Refusals.cs`, and `JoinSettlePlan.cs` split
behavior by presentation category. Top-level `PresentationModules.cs` wires their callbacks.

The presentation signal/record files include `AnimationRecorder.cs`, `AudioRecorder.cs`,
`EventQueueRecorder.cs`, `LookAtRecorder.cs`, `MotionWatch.cs`, `ObjectsRecorder.cs`,
`RouteRecorder.cs`, `SimulatorRecorder.cs`, `SlotsRecorder.cs`, `VfxRecorder.cs`,
`WorldRecorder.cs`, and `WorldRecorderRoadPlace.cs`. Compact wire helpers include
`ConversationState.cs`, `InventorySet.cs`, `MenuRefusal.cs`, `ObjectUse.cs`,
`RelationshipState.cs`, `RouteLegIndex.cs`, `SimAliasWire.cs`, `SimDetailsWire.cs`,
`SimPresenceWire.cs`, `SimScopeWire.cs`, `SimStateWire.cs`, `UserIntentWire.cs`,
`WalkerCopyLine.cs`, and `WalkHaltTiming.cs`.

Native/integration helpers include `SacsChunkReserve.cs`, `SacsFreeRing.cs`, `SacsGc.cs`,
`SacsGcHeap.cs`, `SacsOnScreen.cs`, `SacsOrder.cs`, `SacsPicks.cs`, `SacsRandom.cs`,
`SacsSeedCopy.cs`, and `SacsWalkerCopy.cs`. Their files describe the managed boundary and
counters; use [simlink.md](simlink.md) for the corresponding native component.

## Compatibility and failure cases

* The mod and controller bridge have mirrored message definitions. A verb or field-order
  change must update both codecs and parity tests.
* The bridge `CONNECT`/`HELLO` handshake and build stamp are needed before normal commands.
  Do not infer readiness merely because the game process exists.
* A command can be validly parsed but rejected by role or authority gate. Inspect the
  response and role state; do not treat a successful bridge socket write as successful
  engine application.
* Captured interaction can be intentionally filtered. The host can report a
  dropped/capture-filtered event; absence from the public event stream does not prove that
  the local game did nothing.
* A client-side apply may be deferred until the relevant Sim exists. Position application
  can also be held when a route owns the target; the handler returns a distinct status for
  deferred/route-held paths.
* Runtime tunables disappear on restart. When a settled value should be default behavior, it
  must be changed in source rather than treated as persistent config.
* The optional SimLink path is version/probe-gated. A missing sidecar should not be papered
  over by bypassing the gate.

## How to verify behavior

The main controller test executable source-links pure mod policy files so tests exercise the
same logic used by the mod. Examples include `AutonomyCommandGate.cs`,
`ClientCaptureFilter.cs`, `UnclaimedRoutingEmitFilter.cs`, `OutboundLaneQueue.cs`,
`SelectionGuard.cs`, `InventorySet.cs`, `MenuRefusal.cs`, `ObjectUse.cs`, and
`Snapshot2Writer.cs`. `BridgeParityTests.cs` compares shared layout and CRC behavior across
the two endpoints.

Useful test subsets include `AutonomyCommandGate`, `ClientCaptureFilter`, `BridgeParity`,
`SelectionGuard`, `Snapshot2`, and `Presentation` (filters match case names, not file
paths). Full mod compilation is a separate Windows gate because the engine-coupled files use
legacy references; see [build-and-solution-layout.md](build-and-solution-layout.md) and
[tests-map.md](tests-map.md).

For the controller-side sender and persistence boundary, see
[dedicated-server.md](dedicated-server.md). For public events and DTOs, see
[protocol-library.md](protocol-library.md). For the native transport and wrapper, continue
to [simlink.md](simlink.md).

## Tracing a specific mod command

For a bridge command such as `SET_AUTHORITY`, the source path is:

1. The controller builds `BridgeCommand.SetAuthority` with an `AuthorityGateMode` and owned
   ids.
2. The controller's `BridgeLineCodec` encodes the bridge verb and its arguments.
3. The mod's `BridgeCodec.DecodeCommand` parses the matching line into its command shape.
4. `ServerHostMain` dispatches the verb and applies the role/gate state on the host side.
5. The mod emits a correlated `OK` or `ERR` response; `HostBridgeServer` completes the
   controller's pending send.
6. Session policy decides whether to publish a public event or return only a command result.

For user intent, the direction reverses: the mod's capture path applies the pure allowlist
in `ClientCaptureFilter`, emits an intent event through `BridgeCodec`, the controller
correlates it to its requester, and the session turns it into a public event or a
server-side command. This is why a mod-side hook alone does not determine whether an action
is allowed: the controller also checks joined player identity and ownership.

## Implementation census (tracked files)

The mod directory contains 93 tracked files. This catalog groups all source and packaging
entry points by their interoperability role; it deliberately avoids describing proprietary
game internals.

| Group | Files |
|---|---|
| Build and bootstrap | `README.md`, `SimsOnlineServerHostMod.Ide.csproj`, `build-server-host-mod.ps1`, `install-server-host-mod.ps1`, `package-server-host-mod.ps1`, `Initializer.cs`, `ServerHostMain.cs`, `MethodSwap.cs`, `Sims3PackageBuilder.cs`, `BuildStamp.cs`, `Tunables.cs` |
| Bridge transport | `BridgeCodec.cs`, `BridgeMemoryBuffer.cs`, `BridgeSharedRegionLayout.cs`, `BridgeBulkBuffer.cs`, `BridgeCrc32.cs`, `OutboundLaneQueue.cs`, `EmbeddedHandlerAssembly.cs`, `EmbeddedLoadTestAssembly.cs` |
| Handler subsystem | `Handlers/HandlerSet.cs` |
| Authority and selection | `AutonomyCommandGate.cs`, `OwnedHostDetourFilter.cs`, `ClientCaptureFilter.cs`, `UnclaimedRoutingEmitFilter.cs`, `SelectionGuard.cs`, `ParamAuthority.cs`, `RepairEntryGate.cs`, `RepairLabel.cs` |
| Interaction/routing policy | `InteractionReplayApplySorter.cs`, `RoutingInteractionFilter.cs`, `TerminalInteractionFilter.cs`, `SitTransitionEmitFilter.cs`, `RouteLegIndex.cs`, `RustRoadPlace.cs`, `RustRoutePlan.cs`, `WalkHaltTiming.cs`, `WalkerCopyLine.cs` |
| Replay/catch-up | `BoundaryReplay.cs`, `BoundaryState.cs`, `DriverIntroduction.cs`, `IdleRepairPlan.cs`, `QueueMirrorItem.cs`, `ReplaySetupCache.cs`, `ReplayStateCache.cs`, `StateCatchUp.cs`, `SwallowedErrors.cs` |
| Presentation modules | `PresentationModules.cs`, `Presentation/Module.cs`, `Presentation/Inventories.cs`, `Presentation/JoinSettlePlan.cs`, `Presentation/Moodlets.cs`, `Presentation/Refusals.cs`, `Presentation/Uses.cs`, `Presentation/Walks.cs` |
| Host activity recorders | `AnimationRecorder.cs`, `AudioRecorder.cs`, `EventQueueRecorder.cs`, `LookAtRecorder.cs`, `MotionWatch.cs`, `ObjectsRecorder.cs`, `RouteRecorder.cs`, `SimulatorRecorder.cs`, `SlotsRecorder.cs`, `VfxRecorder.cs`, `WorldRecorder.cs`, `WorldRecorderRoadPlace.cs` |
| State/intent wire helpers | `ConversationState.cs`, `InventorySet.cs`, `JoinFingerprint.cs`, `LineKindCounts.cs`, `MenuRefusal.cs`, `ObjectUse.cs`, `RelationshipState.cs`, `SimAliasWire.cs`, `SimDetailsWire.cs`, `SimPresenceWire.cs`, `SimScopeWire.cs`, `SimStateWire.cs`, `UserIntentWire.cs` |
| Snapshot and native module wrappers | `Snapshot2Writer.cs`, `SimLinkNative.cs`, `SacsChunkReserve.cs`, `SacsFreeRing.cs`, `SacsGc.cs`, `SacsGcHeap.cs`, `SacsOnScreen.cs`, `SacsOrder.cs`, `SacsPicks.cs`, `SacsRandom.cs`, `SacsRecorder.cs`, `SacsSeedCopy.cs`, `SacsWalkerCopy.cs` |

Use the source file itself to identify the exact bridge verb and fields before changing a
listed path. The file groups above are a navigation aid; they do not imply that every
recorder has the same cadence or that every feature is enabled in every host role.
