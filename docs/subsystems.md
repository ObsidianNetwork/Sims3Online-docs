# Subsystem reference

This page connects the main controller subsystems to their public behavior. It is an orientation
map, not a substitute for the source contract or the test that covers a particular path.

## Session listener and command authority

`DedicatedServer/Network/ClientListenerServer.cs` owns the player-facing TCP connection loop.
It decodes bounded JSON lines, associates a successful join with the connection, checks that
later commands speak for the joined player, and sends command results and broadcast events.
The host's local loopback operations are a separate administrative path.

This boundary is important for both troubleshooting and security: a correctly parsed command
can still be rejected by an identity or ownership check. See
[network protocol](protocol.md).

## Authoritative session

`DedicatedServer/Authority/AuthoritativeSession.cs` handles session-level operations and
coordinates them with an `IAuthoritativeSimulationHost`. The host abstraction isolates session
logic from the live game integration, which lets controller tests exercise command behavior
without launching The Sims 3.

Important responsibilities include player join/leave, Sim claim ownership, interaction
requests, speed and time changes, snapshots, world bootstrap, and save checkpoints.

## Snapshot and mirror synchronization

The host game provides snapshots used to initialize and update client mirrors. Client
controllers acknowledge applied ticks. `SyncBroadcaster` and the mirror coordination code
monitor progress and can request replacement from a full snapshot when a delta is no longer an
appropriate catch-up path.

Snapshot state and presentation records are separate streams. This distinction helps diagnose a
client that has correct position/state but missing animation, or whose visual stream looks
plausible while its state snapshot is stale.

## Ownership and lot reconciliation

Sim claims associate players with Sims for command authorization. The separate lot-local mode
supports a client-native owner for selected lot state. In that mode the client exports a
structured lot bundle, the server validates and merges it, and version acknowledgements let the
client track whether its last export was accepted.

The save path checks reconciliation freshness for client-native owners. This couples
persistence safety to ownership synchronization; a save refusal should be debugged as a
reconciliation problem rather than retried blindly.

## Clock reconciliation

The host game is the time authority. The session reads the host clock and emits an absolute
`world_time_changed` event when mirror correction is needed. A joining player also receives the
current time when available. This design periodically corrects drift between independent game
processes without claiming deterministic lockstep.

## Interaction and presentation

The interaction lifecycle can emit queue, start, completion, and failure events. The separate
presentation translator handles selected local-game actions such as animation replay and state
application. The distinction between lifecycle evidence, snapshot data, and visual output is
described in [simulation and presentation](simulation.md).

## Source navigation

| Question | Start here |
|---|---|
| What commands can a player send? | `SimsOnline.Protocol/ClientCommand.cs` |
| What does the server emit? | `SimsOnline.Protocol/ServerEvent.cs` |
| Which player may issue a command? | `DedicatedServer/Network/ClientListenerServer.cs` |
| How are session operations handled? | `DedicatedServer/Authority/AuthoritativeSession.cs` |
| How are client events applied? | `DedicatedServer/Mirroring/EventToBridgeCommandTranslator.cs` and `MirrorEngine.cs` |
| How is the local bridge encoded? | `DedicatedServer/HostBridge/BridgeLineCodec.cs` |
| How is binary snapshot data decoded? | `DedicatedServer/HostBridge/Snapshot2BinaryReader.cs` |

When publishing a technical claim, link it to a test or a reproducible scenario whenever
possible. Source organization can change; update these pointers when modules move.
