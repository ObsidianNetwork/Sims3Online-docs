# Local bridge and data formats

This page describes the two formats most useful for understanding controller/game integration:
the public TCP envelope and the local host bridge. The exact codecs in the current source are
the implementation authority.

## Public TCP messages

Each public message is a UTF-8 JSON object followed by a newline. The serializer uses camel-case
property names, string enum values, and a `$type` discriminator. See
[network protocol](protocol.md) for message categories and [the protocol source](https://github.com/ObsidianNetwork/Sims3Online/blob/main/SimsOnline.Protocol/ClientCommand.cs)
for exact fields.

The serializer represents a JSON `byte[]` as a Base64 string. This is used by messages such as
`sim_description_data` and `world_file_chunk`; do not confuse the Base64 text in JSON with the
underlying binary game data.

## Host bridge

The game mod and dedicated server share a local control region and a bulk-data region. Their
layout definitions are mirrored in `SimsOnlineServerHostMod/BridgeSharedRegionLayout.cs` and
`DedicatedServer/HostBridge/BridgeSharedRegionLayout.cs`.

Current source constants:

| Property | Current value |
|---|---:|
| Layout version | 11 |
| Bridge protocol version | 23 |
| Hello/capability wire version | 6 |
| Control slots | 16 total, split into 8 slots per direction |
| Control slot size | 4,096 bytes, including a 16-byte slot header |
| Control payload limit | 4,080 bytes |
| Bulk slots | 4 |
| Bulk slot size | 524,288 bytes, including a 32-byte slot header |
| Bulk payload limit | 524,256 bytes |

The control ring carries framed commands and responses. Larger snapshots or serialized payloads
use a bulk slot, with explicit producer/consumer state so the payload is not mistaken for a
complete message before publication. The control codec also carries verb-specific text fields;
use the shared `BridgeLineCodec` rather than constructing frames by hand.

These constants describe the local implementation, not the public TCP protocol. When changing a
layout or bridge verb, keep both sides in sync and update the bridge contract tests.

## SNAPSHOT2 binary snapshot

The in-game mod writes the payload in `SimsOnlineServerHostMod/Snapshot2Writer.cs`; the
controller decodes it in `DedicatedServer/HostBridge/Snapshot2BinaryReader.cs`. The format is
little-endian and uses a 32-bit unsigned byte length followed by UTF-8 bytes for each string.
The top-level record is:

| Order | Field | Encoding |
|---:|---|---|
| 1 | Magic | `u32`, current numeric value `0x32505353` |
| 2 | Format version | `u32`; reader accepts version 1 and 2 |
| 3 | Simulation tick | `i64` |
| 4 | World name | length-prefixed UTF-8 |
| 5 | Save name | length-prefixed UTF-8 |
| 6 | Game-speed ordinal | `u32` |
| 7 | Sim count | `u32` |
| 8 | Sim records | repeated `Sim count` times |
| 9 | Object count | `u32` |
| 10 | Object records | repeated `Object count` times |

A Sim record contains the object ID, name, household ID, three `f32` position coordinates,
current interaction ID and key, owning-player ID, lot ID, interaction-queue count, posture
container ID and type, and—starting in version 2—the Sim-description ID. An object record
contains object ID, script class, three position coordinates, and lot ID.

Zero represents an absent nullable `u64`; `0xFFFFFFFF` represents an absent nullable `u32`.
The nullable-string reader treats a zero byte length as null; ordinary strings use an empty
string for zero length. Version 2 adds the description ID at the end of each Sim record so
older layout data is not reinterpreted as a newer field.

The decoder checks the magic, supported version, record lengths, and required fields and
returns no snapshot when decoding fails. This binary format is for local controller/game
communication; it is not a public save-file format and should not be used as a substitute for
the game's save format.

## Game save files

World bootstrap transfers game save files as opaque chunks. The controller tracks file names,
indices, offsets, lengths, and byte payloads; it does not define or decode the proprietary Sims 3
save-file format. Compatibility and integrity of those files remain the game's responsibility.

## Safe format examples

Use synthetic examples such as a snapshot with an empty world name and zero Sims. Never commit
real player names, object IDs, save chunks, captured game bytes, or extracted game assets.
