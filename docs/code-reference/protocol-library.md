# Protocol library

`SimsOnline.Protocol` is the C# contract assembly shared by the dedicated controller, the
client, the test client, and .NET tests. The project is deliberately DTO/codec-centered: it
defines public commands and events, snapshots, transfer payloads, and validation rules. It
does not own TCP sockets, session authorization, retries, or host-bridge state.

The library is a separate project (`SimsOnline.Protocol/SimsOnline.Protocol.csproj`) with
assembly name `SimsOnline.Protocol`. Its root namespace remains
`SimsOnline.DedicatedServer.Protocol` for compatibility with the controller types. The
controller, client, test client, and tests reference it directly; see
[build-and-solution-layout.md](build-and-solution-layout.md).

## Two public wire encodings

The project currently contains two protocol representations:

| Transport | Entry point | Framing | Discriminator / versioning |
|---|---|---|---|
| JSON line | `PublicProtocolCodec` | One UTF-8 JSON object per newline | `$type` with stable snake-case names such as `join_session` and `session_joined`; JSON codec has no separate numeric version handshake. |
| Line v1 | `PublicLineCodecV1` | Pipe-separated UTF-8 fields, one frame per line | `HELLO` / `WELCOME` protocol version; `PUBLIC_PROTOCOL_VERSION` is `1`. |

Do not treat these as interchangeable serializers. JSON uses camel-case property names,
camel-case enum strings, and omits nulls. Line v1 encodes record constructor fields as
positional values; it hex-encodes strings and byte arrays and represents nullable values as
`~`. A field-order change can break line-v1 even when a JSON property addition is
compatible.

### JSON command example

This illustrates the JSON names produced by `PublicProtocolCodec.JsonOptions` for a join
command. `OnScreen` and diagnostic fields are null in this example and therefore omitted.

```json
{"$type":"join_session","commandId":"00000000-0000-0000-0000-000000000001","playerId":1,"displayName":"probe"}
```

The codec sets `JsonNamingPolicy.CamelCase`, a camel-case dictionary-key policy,
`JsonIgnoreCondition.WhenWritingNull`, and `WriteIndented=false`. Enums use
`JsonStringEnumConverter(JsonNamingPolicy.CamelCase)`. Deserialization is polymorphic on
`$type`; `TryDecodeCommand` catches JSON, unsupported-type, and format-level failures and
returns an error string rather than throwing for those cases.

### Line-v1 handshake and frames

A client can declare capabilities in the handshake:

```text
HELLO|1|ring=0,wfs=0,role=client
WELCOME|1|7|<capabilities>
```

`HELLO` fields are `tag|version|caps`. `WELCOME` fields are `tag|version|epoch|caps`. A
response may then contain a resume result if the client presented resume credentials.
Example command line for a `JoinSession` with `DisplayName="probe"` and null `OnScreen`:

```text
CMD|1|JoinSession|1|1|70726f6265|~
```

The frame fields are `CMD|corr|verb|commandId|playerId|args…`. The `70726f6265` field is
lowercase hex UTF-8 for `probe`; `~` is null. A minimal event envelope is
`EV|seq|tick|corr|kind|fields…`; the codec appends the fields from the event record. Text
fields are hex-encoded; integers use invariant decimal; booleans are `0`/`1`; enums use
their declared names; floats reject non-finite values.

The encoder rejects pipe/newline characters in outer fields and rejects a line over
`PUBLIC_LINE_MAX_BYTES = 4,194,304` UTF-8 bytes. `Split` accepts one trailing LF and
optional CR, but embedded CR/LF is an error. It uses strict UTF-8 decoding rather than
replacing invalid byte sequences.

## Command contract

`ClientCommand` is an abstract record with common `CommandId` (`Guid`) and `PlayerId`
(`int`) fields. JSON discriminator names are explicit attributes. The currently registered
command records are:

| Record | Purpose / important fields |
|---|---|
| `JoinSession` | Join player by display name; optional `OnScreen` report. |
| `LeaveSession` | Leave and trigger claim cleanup. |
| `ClaimSim`, `ReleaseSim` | Claim or release a Sim object id. |
| `SetActiveSim` | Select active Sim for the player. |
| `QueueInteraction` | Submit an `InteractionRequest` with actor, interaction key, optional target id/position, and string parameters. |
| `CancelInteraction` | Cancel a specific actor/interaction id. |
| `LeaveConversation` | Leave conversation for an actor. |
| `AnswerDialog` | Answer a request id for a Sim; nullable answer means no explicit answer. |
| `RequestSimDescription` | Fetch missing description payload by description id. |
| `RequestWorld` | Ask server for the checkpointed world before local game load; it does not require `JoinSession`. |
| `SetGameSpeed` | Set a `GameSpeed` enum value. |
| `SetWorldTime` | Set day/hour/minute as `WorldTime`. |
| `RequestSnapshot` | Request full/delta snapshot; nullable `SinceTick` indicates no delta cursor. |
| `SetClientRole` | Declare `PlayerRole.Server` or `PlayerRole.Client`; session requires join first. |
| `SaveGame` | Ask the server to checkpoint a named save. |
| `AckSnapshot` | Acknowledge the simulation tick actually applied by a mirror. |
| `SubmitLotExport` | Submit lot-scoped active-lot state with last acknowledged reconciliation version. |

The role declaration and host role are separate domains. `PlayerRole` describes a connected
public client; the in-game mod has its own bridge-side role. Do not treat these as a single
authority bit.

`InteractionRequest` is the structured input shape. `UserPick` reserves `ui.name`,
`ui.path`, `ui.hit`, and `ui.picked`; `IsUserPick` checks for the `ui.name` parameter.
`GameSpeed` values are `Pause`, `Normal`, `Speed2`, `Speed3`, and `Ultra`. `WorldTime`
stores day/hour/minute; its `Hours24` property produces `Hours + Minutes / 60f` for the host
setter.

### Command example with a typed pick

The command keeps the interaction and user-pick fields in one request. The actor/target
identifiers are server-side object ids, not display names.

```json
{
  "$type":"queue_interaction",
  "commandId":"00000000-0000-0000-0000-000000000002",
  "playerId":1,
  "interaction":{
    "actorObjectId":100,
    "interactionKey":"Terrain.GoHere",
    "targetObjectId":null,
    "targetPosition":{"x":4,"y":0,"z":9},
    "parameters":{"ui.name":"Go Here"}
  }
}
```

The specific parameters are not a promise that every named interaction is valid for every
actor. The session and host perform ownership, target, queue, and engine validation; the
protocol carries the request and its response.

## Event contract

`ServerEvent` is the server-to-client polymorphic hierarchy. It includes connection/command
results, ownership and interaction lifecycle, snapshots, clock/speed, persistence and state
changes, and presentation data.

| Event family | Examples | Use |
|---|---|---|
| Join/command outcome | `SessionJoined`, `CommandAccepted`, `CommandRejected`, `ServerError` | Explain whether an operation/session was accepted. |
| Ownership and roles | `SimClaimed`, `SimReleased`, `ClientRoleChanged`, `LotLocalAuthorityModeChanged` | Reflect current player control/authority state. |
| Interaction lifecycle | `InteractionQueued`, `InteractionReplayQueued`, `InteractionStarted`, `InteractionCompleted`, `InteractionFailed` | Queue and completion feedback. |
| Captured intent | `IntentCaptured`, `UserIntentCaptured`, `SpeedIntentCaptured`, `DialogAnswerCaptured`, `CancelIntentCaptured`, `LeaveConversationIntentCaptured` | Report input observed by the host. |
| State/snapshot | `FullSnapshot`, `SnapshotDelta`, `SimStateChanged`, `SimMotiveChanged`, `SimDetailsChanged`, `SimSpawned`, `SimDespawned` | Synchronize world and Sim state. |
| Clock and save | `GameSpeedChanged`, `WorldTimeChanged`, `SaveCheckpointed`, `WorldFileChunk` | Synchronize time and transfer save files. |
| Presentation | `SimAnimation`, visible-state and presence events | Present authoritative activity in a client. |
| Lot reconcile | `LotExportRequested`, `LotReconcileAcknowledged` | Request a lot payload and return its reconciled version. |

`SimulationSnapshot` has `Tick`, `WorldName`, `SaveName`, `Speed`, `Sims`, and `Objects`.
`SimSnapshot` includes `ObjectId`, `Name`, `HouseholdId`, `Position`, optional current
interaction, optional owner/lot, optional queue count, posture fields, and
`SimDescriptionId`. `ObjectSnapshot` carries object id, script class, position, and optional
lot id. `Vector3Dto` is an X/Y/Z record struct. Optional fields allow an older mod to omit
data such as Sim description id.

A delta is still a `SimulationSnapshot`; the event distinguishes full (`FullSnapshot`) from
`SnapshotDelta` and may request a full reset through `ForceFull`. The client must preserve
event ordering and only acknowledge ticks it applied.

## Line-v1 limits, resume and bulk

`PublicLineCodecV1` centralizes line parsing, field conversion, frame validation, and event
assembly. Its public constants and validation ceilings are:

| Constant / bound | Value | Meaning |
|---|---:|---|
| `PUBLIC_PROTOCOL_VERSION` | `1` | Version in HELLO/WELCOME. |
| `PUBLIC_LINE_MAX_BYTES` | `4,194,304` bytes | Maximum encoded line. |
| `PUBLIC_BULK_CHUNK_RAW_BYTES` | `3,145,632` bytes | Raw bytes per bulk chunk encoder. |
| `WORLD_CHUNK_RAW_BYTES` | `WorldSave.ChunkBytes` | Raw bytes per world-file chunk (256 KiB). |
| `ResumeOkFrame.Count` | at most `2048` | Upper bound accepted by `DecodeResume`. |
| `BulkReceiver` default total | `256 * 1024 * 1024` | Maximum bulk bytes tracked by default. |
| `BulkReceiver` default in flight | `16` | Maximum simultaneous assemblies by default. |

A resume token is exactly 32 lowercase hexadecimal characters. `ResumeToken` only recognizes
the `JoinSession` acknowledgement and rejects duplicate or malformed `resume_token=` fields.
`ResumeOkFrame` requires nonzero `FromSeq` and a bounded replay `Count`;
`PublicWireConnection` additionally checks epoch and that replay begins at cursor sequence +
1. `ResumeFreshFrame` resets connection-local event and command maps.

Bulk traffic uses `BULK|bulkId|index|count|base64`. `BulkReceiver.Introduce` announces a
bulk id before chunks arrive; `TrySubmit` validates shape, count, base64, order, and
configured size limits. An `EV` whose payload is bulk-backed is not released to the client
until its associated bulk bytes are complete. Ack bodies can also be emitted as bulk chunks
via `EncodeAckLines`.

World transfer is a different semantic layer over bulk: `WorldSave.ToChunks` emits one
`WorldFileChunk` per file segment in file order. `WorldSaveWriter` validates save
folder/file names, file index progression, and exact byte offsets; it stages all files and
only promotes the folder once complete. The separate marker prevents overwriting a user's
unrelated save with the same name.

## Lot export and validation

`LotExport.cs` defines client-to-server lot reconciliation payloads: `LotSimExport`,
`HouseholdExportPayload`, `InventoryExportPayload`, `LotExportBundle`, and result/rejection
records. The API exposes `HouseholdExportValidator.Validate` and the
`ObjectGuidRemapRequiresProof` rule. The current constant is true: object-id remapping
cannot be assumed safe without proof.

The public type surface deliberately limits scope. A lot export is for owned active-lot
state. It is not a general full-world save or a replacement for server authority over global
state. `ClientCommand.SubmitLotExport` carries the last `LotReconcileAcknowledged` version
the client received; the session uses that version in conflict classification. See
`DedicatedServer/Mirroring/LotReconcile.cs` and
`AuthoritativeSession.TryDeepMergePendingLotExportAsync` for integration.

`SimDescriptionPayload` groups a description id and three byte arrays (`PropertyStream`,
`ResKeyTable`, `ObjectIdTable`). `Combined`, `FromCombined`, and `Crc32` are the helper
surface for combined transfer framing. `SimBuffDto` is the small `(Guid, Origin)` wire
record.

## Versioning and compatibility rules

1. Keep the JSON discriminator strings stable. The attributes on `ClientCommand` and
   `ServerEvent` are the explicit JSON registry.
2. For line v1, keep record constructor parameter order stable. The encoder reads the
   positional constructor and serializes each field in order; property order is irrelevant.
3. For protocol-level incompatible changes to line framing, use a version change rather than
   silently reinterpreting a field.
4. Add nullable fields only with deliberate default/compatibility handling. `Construct`
   checks nullable reference annotations and rejects unexpected null.
5. Keep line delimiters and composite encoding unambiguous. Strings are hex rather than
   escaped into the separator grammar; composite fields are parsed with balanced
   brackets/parentheses.
6. Reject unsupported enum strings, malformed hex, non-finite floats, invalid UTF-8,
   unsigned negative integers, and wrong field counts. Do not normalize these invalid inputs
   into defaults.
7. Update JSON pinning and line conformance tests when changing the contract.

`RefusalCode` enumerates structured errors including `UnauthorizedClientInsertion`,
`ClientWorldServiceSuppressed`, `NativeOracleOwned`, `HelloRejected`, `BadLine`,
`NotJoined`, `WrongPlayer`, `Superseded`, `LineTooLong`, `BulkMalformed`, `SimNotFound`,
`TargetNotFound`, `QueueRejected`, and `Dialog`. `RefusalToken` maps selected values to
stable snake-case tokens (for example `hello_rejected`, `bad_line`, `queue_rejected`,
`line_too_long`, `bulk_malformed`); the rest use enum names. `ParseRefusal` accepts only
values in that mapping.

## File inventory

| File | Main responsibility |
|---|---|
| `ClientCommand.cs` | Command records, `PlayerRole`, `UserPick`, `InteractionRequest`, `GameSpeed`, `WorldTime`. |
| `ServerEvent.cs` | Event record hierarchy and event discriminators. |
| `PublicProtocolCodec.cs` | Compact JSON encode/decode and serializer settings. |
| `PublicLineCodecV1.cs` | Versioned line framing, conversion, correlation mapping, bulk and event receivers. |
| `SimulationSnapshot.cs` | Snapshot, Sim/object rows, position DTO. |
| `ServerCapability.cs` | Capability lookup helper. |
| `LotLocalAuthorityMode.cs` | Wire-carried lot authority enum. |
| `LotExport.cs` | Lot, household and inventory export types and validation. |
| `WorldSave.cs` | Save-folder read, chunk generation and safe client-side staging writer. |
| `SimDescriptionPayload.cs` | Split/combined Sim description payload and CRC. |
| `SimBuffDto.cs` | Buff GUID/origin pair. |
| `InteractionReplayBundle.cs` | Interaction replay record and replay policy/queue role enums. |
| `SimsOnline.Protocol.csproj` | Assembly identity and project settings. |

## Verification

`DedicatedServer.Tests/WireJsonPinningTests.cs` locks JSON wire output;
`PublicLineCodecTests.cs` checks all registered kinds and conformance fixtures;
`ClientWireLineTests.cs` exercises handshake, resume, join, bulk, snapshots, and world fetch
through the client. `ProtocolHardeningTests.cs` covers malformed or oversized inputs and
boundary validation.

The executable test runner supports a case-name substring filter:

```powershell
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter PublicLineCodec
```

This selects `PublicLineCodec`-named cases; it does not select every test that happens to
use protocol types. See [tests-map.md](tests-map.md) for additional subsets and
[testing](../testing.md) for the repository's aggregate commands. For the controller that
consumes these records, continue to [dedicated-server.md](dedicated-server.md); for
client-side line conversion and queueing, see
[client-and-test-client.md](client-and-test-client.md).
