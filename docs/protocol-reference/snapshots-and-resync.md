# Snapshots and resynchronization

`SimulationSnapshot` is the server's world-state model on the public JSON path and the decoded result of the process-memory snapshot bridge. The same record can be serialized as JSON or received through the binary SNAPSHOT2 payload; those encodings are not interchangeable.

## JSON snapshot model

| Record | Fields in declaration order |
|---|---|
| `SimulationSnapshot` | `Tick: long`, `WorldName: string`, `SaveName: string`, `Speed: GameSpeed`, `Sims: IReadOnlyList<SimSnapshot>`, `Objects: IReadOnlyList<ObjectSnapshot>` |
| `SimSnapshot` | `ObjectId: ulong`, `Name: string`, `HouseholdId: ulong`, `Position: Vector3Dto`, `CurrentInteractionId: ulong?`, `OwningPlayerId: int?`, then optional `CurrentInteractionKey`, `LotId`, `InteractionQueueCount`, `PostureContainerObjectId`, `PostureType`, `SimDescriptionId` |
| `ObjectSnapshot` | `ObjectId: ulong`, `ScriptClass: string`, `Position: Vector3Dto`, `LotId: ulong?` |
| `Vector3Dto` | `X: float`, `Y: float`, `Z: float` |

`SnapshotDelta` wraps a `SimulationSnapshot` plus `ForceFull` (default `false`). The snapshot may carry changed records only. When `ForceFull=true`, the receiver must replace its mirror state with that payload, not fold it as an ordinary delta. `FullSnapshot` is a separate event for an explicit full snapshot request. `AckSnapshot.Tick` acknowledges the tick the client actually applied.

## Ack lag policy

`SyncBroadcaster` computes each player's lag from the broadcast tick and that player's own last acknowledged tick. It does not use a global minimum acknowledgement. `ResyncPolicy.Decide` implements this default table:

| Ack state / lag | Action |
|---|---|
| No valid ack baseline | Publish delta at base interval; do not force a full snapshot at tick zero. |
| `lag <= 1,500` ticks | Publish delta at base interval. |
| `1,500 < lag <= 6,000` | Publish, linearly stretching the interval up to 10,000 ms. |
| `lag > 6,000`, no full in flight | Send one `SnapshotDelta{ForceFull=true}`. |
| `lag > 6,000`, full still unacknowledged | Suppress further broadcasts until the acknowledgement clears the in-flight latch. |

Defaults are `WarnTicks=1500`, `HardTicks=6000`, and `MaxIntervalMs=10000`; session configuration can override them. A stale ack does not clear a newer full-resync latch. The client-side mirror fold limit defaults to 8, and the snapshot-resync request cooldown defaults to 5 seconds.

## SNAPSHOT2 binary layout

The mod writes SNAPSHOT2 directly to a byte buffer and sends it through bridge bulk slots. The format is little-endian. Lengths count UTF-8 bytes, not characters. Strings are encoded as `u32 byteLength` followed by that many bytes; a zero length is empty/null depending on the field. There is no padding between fields.

| Order | Type | Field / meaning |
|---:|---|---|
| 0 | `u32` | Magic constant `0x32505353` |
| 4 | `u32` | Format version: writer emits 2; reader accepts 1 or 2 |
| 8 | `i64` | Tick |
| 16 | `u32` + bytes | World name length and UTF-8 bytes |
| variable | `u32` + bytes | Save name length and UTF-8 bytes |
| variable | `u32` | Speed ordinal: 0 Pause, 1 Normal, 2 Speed2, 3 Speed3, 4 Ultra |
| variable | `u32` | Sim count |
| repeated | record | Sim entries, in order below |
| variable | `u32` | Object count |
| repeated | record | Object entries, in order below |

Each Sim record is: `u64 objectId`; length-prefixed UTF-8 `name`; `u64 householdId`; three `f32` position coordinates; `u64 currentInteractionId` (0 means null); length-prefixed `interactionKey` (zero length means null); `u32 owningPlayerId` (`0xffffffff` means null); `u64 lotId` (0 means null); `u32 interactionQueueCount` (`0xffffffff` means null); `u64 postureContainerObjectId` (0 means null); length-prefixed `postureType` (zero length means null); and, in version 2, `u64 simDescriptionId` (0 means null). Version 1 omits the final `simDescriptionId` field.

Each object record is: `u64 objectId`; length-prefixed UTF-8 `scriptClass`; three `f32` position coordinates; `u64 lotId` (0 means null). Version 2 preserves the `SimDescriptionId` used to correlate server object identity with a client's instance.

**Magic spelling discrepancy:** the source comments call the magic "SPS2", but the shared constant is `0x32505353`. Stored little-endian, its bytes are `53 53 50 32` (`SSP2`). Both writer and reader use the numeric constant; interop should pin the numeric value/bytes, not the comment mnemonic.

The reader rejects a wrong magic, unsupported version, truncated fixed fields, or string lengths beyond the remaining buffer and returns null on parse failure. The writer emits version 2. `DedicatedServer\HostBridge\HostBridgeServer.cs` detects the magic in a completed bulk payload before attempting text decoding.

### Worked example for an empty version 2 snapshot

`Snapshot2.RoundTripNullableFields` in `DedicatedServer.Tests/Program.cs` uses the
mod's writer and the controller's reader. Its complete 36-byte payload starts at byte
zero of the bridge bulk payload, after the transport metadata has been removed.

| Byte offset (decimal) | Bytes (hex) | Decoded value |
|---:|---|---|
| 0 | `53 53 50 32` | Magic `0x32505353` |
| 4 | `02 00 00 00` | Format version 2 |
| 8 | `63 00 00 00 00 00 00 00` | Tick 99 as a little-endian `i64` |
| 16 | `00 00 00 00` | World name has zero UTF-8 bytes |
| 20 | `00 00 00 00` | Save name has zero UTF-8 bytes |
| 24 | `00 00 00 00` | Game speed `Pause` |
| 28 | `00 00 00 00` | Zero Sim records |
| 32 | `00 00 00 00` | Zero object records |

Each empty name uses just its four-byte length. The object count immediately follows
the Sim count because there are no Sim records. `Snapshot2.RoundTripSimple` covers
populated records, including a version 2 Sim description ID. `Snapshot2.DecodesVersion1WithoutSimDescriptionId`
checks that the reader can also decode the shorter version 1 Sim record.

## Verification

`DedicatedServer.Tests\Program.cs` includes the `Snapshot2BinaryReader` and JSON snapshot tests; `DedicatedServer.Tests\ClientWireLineTests.cs` exercises event chunks and large bulk-backed data. Use the current test runner in [testing](../testing.md) rather than hand-editing binary bytes. `ResyncPolicy` decisions are pure and can be verified independently of sockets in the dedicated-server tests.

Related pages: [transport limits](transport-limits.md), [wire formats](wire-formats.md), [bridge layout](bridge-layout.md).
