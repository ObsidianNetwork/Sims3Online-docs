# Protocol and transport reference

This folder is the implementation-oriented reference for the protocol boundaries in the repository. It separates the established .NET JSON client socket, the experimental Rust line-v1 socket, the process-memory bridge, and the optional SimLink transport. Field inventories and wire examples are derived from current types and codecs rather than old prose.

| Page | Answers |
|---|---|
| [Wire formats and compatibility](wire-formats.md) | Which transport is speaking, how JSON and line-v1 encode values, and what a version bump means. |
| [Client command catalogue](command-catalog.md) | Every JSON command discriminator, constructor field/type, and example. |
| [Server event catalogue](event-catalog.md) | Every registered JSON event discriminator, field/type, and example. |
| [Handshake and resume](handshake-and-resume.md) | What `HELLO` proves, how capabilities are checked, and when warm reconnect is safe. |
| [Errors and refusals](errors-and-refusals.md) | Exact line-v1 refusal tokens and how they differ from JSON command outcomes. |
| [Transport limits](transport-limits.md) | Line, bulk, queue, and timeout limits from code. |
| [Snapshots and resynchronization](snapshots-and-resync.md) | Snapshot records, acknowledgement lag, `forceFull`, and the SNAPSHOT2 byte layout. |
| [Evidence JSON schemas](schema-reference.md) | All 39 evidence schema root models plus the four example payloads. |
| [Bridge layout and versions](bridge-layout.md) | The v11 shared-memory region, control ring, bulk slots, and independent version numbers. |
| [Bridge verbs](bridge-verbs.md) | The 75 verbs and which current controller/mod component owns each. |
| [Bridge diagnostics](bridge-diagnostics.md) | Diagnostic command shapes, report envelopes, and development-only operations. |
| [World transfer and lot reconciliation](world-transfer-and-reconcile.md) | Save chunks, safe staging, lot authority modes, and export validation. |
| [Replay, input, and presentation](replay-presentation.md) | Interaction replay bundles, forwarded user picks, remote dialogs, and animation streaming. |
| [SimLink sidecar](simlink.md) | The local native shared-memory record ring and its lifecycle. |
| [Latency and verification](latency-and-tests.md) | What timing paths measure and which tests pin the boundaries. |

## Source of truth

The public JSON model lives in `SimsOnline.Protocol\ClientCommand.cs`, `SimsOnline.Protocol\ServerEvent.cs`, and `SimsOnline.Protocol\PublicProtocolCodec.cs`. The public line-v1 codec is `SimsOnline.Protocol\PublicLineCodecV1.cs`; the Rust line server is under `rust\simlink_headless_server\src`. Process-memory bridge constants are in `DedicatedServer\HostBridge\BridgeSharedRegionLayout.cs` and are mirrored by `SimsOnlineServerHostMod\BridgeSharedRegionLayout.cs`.

The JSON client socket has no protocol version field. The line-v1 handshake is explicitly versioned. Neither is the same version space as the in-process bridge or SimLink. Use the page for the specific channel under test, and do not infer one channel's compatibility from another's.
