# Glossary

Terms below describe distinct ownership, transport, and state concepts. In particular, a
successful command response, a completed game interaction, and a client-rendered result are
different observations.

## Session and protocol

| Term | Meaning |
|---|---|
| **Authoritative game** | The host-side Sims 3 process whose outcomes define shared simulation in the default operating mode. It is not the same process as the dedicated-server controller. |
| **Client controller** | The .NET process that sends player commands to the session and applies server updates to that player's local game. |
| **Command** | A client-to-server request, such as joining, claiming a Sim, or queueing an interaction. Its `commandId` correlates the command result. |
| **Event** | A server-originated report, including command results, state changes, or presentation updates. |
| **Command accepted** | The session handler accepted a command request. It does not mean the underlying game action started or completed. |
| **Joined connection** | A public TCP connection associated with one player after a successful `join_session`. The listener checks that subsequent commands use that player's identity. |
| **JSON protocol** | The established .NET client/server protocol: newline-delimited JSON over TCP, encoded by `PublicProtocolCodec`. |
| **Line-v1** | A separately versioned, experimental line protocol used by the Rust front door. It is not wire-compatible with the .NET JSON protocol. |

See [network protocol](protocol.md), [wire formats](protocol-reference/wire-formats.md), and
[errors and refusals](protocol-reference/errors-and-refusals.md).

## Simulation and authority

| Term | Meaning |
|---|---|
| **Sim claim** | The server-side association between a player and a Sim. The claim is part of command authorization; it is not merely a client selection highlight. |
| **Interaction lifecycle** | The sequence from queue request through game-side start and terminal completion or failure. A transport acceptance is only an earlier step. |
| **Client-native lot authority** | An explicit mode in which client-owned lot data is exported and reconciled under server validation; it is distinct from the default host-authoritative simulation. |
| **Lot reconciliation** | Validation and merging of client-native lot exports, including version acknowledgement where that mode is active. |

See [architecture](architecture.md), [authority and ownership](architecture.md#authority-and-ownership),
and [interactions and lifecycle](simulation-reference/interactions-and-lifecycle.md).

## State and presentation

| Term | Meaning |
|---|---|
| **Snapshot** | A state observation at a simulation tick. A full snapshot supplies a baseline; a delta supplies an update relative to an earlier observation. |
| **Snapshot acknowledgement** | The tick a client reports as applied. The server uses per-player acknowledgement progress to decide whether to publish normally, slow updates, or send a full resync. |
| **Forced full resync** | A full replacement snapshot carried by `snapshot_delta` with `forceFull: true`; the client replaces its mirror instead of applying it as an ordinary delta. |
| **Mirror** | The client's local representation of selected server-originated simulation state. It is not a second authority for default-mode shared outcomes. |
| **Presentation stream** | Selected visual operations, such as animation replay, sent separately from the authoritative simulation snapshot. |
| **World bootstrap** | The transfer of the host's world to a client before it participates in the live session. It is separate from the ongoing snapshot stream. |

See [snapshots and resync](protocol-reference/snapshots-and-resync.md),
[presentation](presentation/index.md), and [world transfer](protocol-reference/world-transfer-and-reconcile.md).

## Local integration and data

| Term | Meaning |
|---|---|
| **Host bridge** | The local controller/game boundary used to issue commands, receive observations, and transfer payloads. It is not the player-facing TCP protocol. |
| **Control ring** | Fixed-size local bridge slots used for framed command and response traffic. |
| **Bulk slot** | A larger local shared-memory region for payloads too large for a control frame. |
| **SNAPSHOT2** | A versioned little-endian binary snapshot representation exchanged between the host game integration and controller. |
| **Checkpoint save** | A controller-managed save boundary used for recovery or world transfer. Checkpoint, join, and last-good saves have separate roles. |

See [bridge and formats](bridge-and-formats.md) and [data and persistence](data-and-persistence/index.md).
